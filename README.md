# proj10 IaC Repository

Infrastructure-as-code for the proj10 MLOps system. This repo covers full lifecycle:

1. Lease/provision OpenStack instances + network.
2. Bootstrap Kubernetes (Kubespray).
3. Deploy platform/pipeline/apps/workflows/secrets.
4. Restore data and operational state.

Use this runbook top-to-bottom for a fresh environment or cluster rebuild.

## 1) Prerequisites

- Terraform 1.x
- Ansible (`ansible-core` compatible with your Kubespray version)
- OpenStack CLI access to Chameleon
- SSH key access to all nodes as `cc`
- `kubectl`, `helm`, `argocd` CLI available where you run commands

Repo paths used below assume:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
```

## 2) Lease + Reservation Setup

Create/activate your Chameleon lease(s), then get reservation UUIDs (not lease IDs):

```bash
openstack lease show <lease-name> -f json | jq
```

Set reservation IDs in `tf/kvm/secrets.auto.tfvars`:

```hcl
suffix      = "<net-or-env-suffix>"
key         = "id_rsa_chameleon"
reservation = "<reservation_id_for_default_nodes>"

# Optional: per-node override (example: node4 on larger flavor reservation)
node_flavor_id_overrides = {
  node4 = "<reservation_id_for_node4>"
}
```

## 3) Provision Network + Instances (Terraform)

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/tf/kvm
terraform init
terraform plan
terraform apply
```

Get floating IP:

```bash
terraform output -raw floating_ip_out
```

## 4) Inventory + SSH Wiring

The repo has two inventories:

- `ansible/inventory.yml` (direct internal IPs).
- `ansible/k8s/inventory/mycluster/hosts.yaml` (Kubespray inventory).

Important: `ansible/k8s/inventory/mycluster/hosts.yaml` currently uses:

```yaml
ansible_host: node1
```

So you must either:

1. Ensure SSH config aliases `node1..node4` resolve correctly.
2. Or replace `ansible_host` with direct IPs.

Quick validation:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible -i ./inventory.yml all -m ping -u cc
ansible-inventory -i ./k8s/inventory/mycluster/hosts.yaml --graph
```

## 5) Bootstrap Kubernetes Cluster

Run from repo root:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ./inventory.yml ./pre_k8s/pre_k8s_configure.yml -b -u cc --ask-vault-pass
ansible-playbook -i ./k8s/inventory/mycluster/hosts.yaml ./k8s/kubespray/cluster.yml -b -u cc
ansible-playbook -i ./inventory.yml ./post_k8s/post_k8s_configure.yml -b -u cc --ask-vault-pass
ansible-playbook -i ./inventory.yml ./ops/dedicate_ml_node.yml -b -u cc --ask-vault-pass
```

`dedicate_ml_node.yml` is required for workflows that schedule with:

- `nodeSelector: workload=large`
- toleration for `dedicated=large:NoSchedule`

If this step is skipped, ML/training workflow pods can remain `Pending` with scheduling errors.

Scale-out (example: add one node):

```bash
ansible-playbook -i ./inventory.yml ./pre_k8s/pre_k8s_configure.yml -b -u cc --limit node4
ansible-playbook -i ./k8s/inventory/mycluster/hosts.yaml ./k8s/kubespray/scale.yml -b -u cc --limit node4
```

## 6) Install Sealed Secrets + Generate/Apply Secrets

### 6.1 Install controller in cluster (on node1)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install sealed-secrets bitnami/sealed-secrets \
  -n kube-system \
  --create-namespace \
  --set-string fullnameOverride=sealed-secrets-controller
kubectl -n kube-system rollout status deploy/sealed-secrets-controller
```

### 6.2 Prepare + seal locally

Use `.env` in repo root as source values.

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ./inventory.yml ./secrets/prepare_k8s_secrets.yml --ask-vault-pass

# 0) In remote node, copy the kubeconfig and change permission for scp
sudo cp /etc/kubernetes/admin.conf /home/cc/admin.conf && sudo chown cc:cc /home/cc/admin.conf && chmod 600 /home/cc/admin.conf

# 1) keep tunnel open (terminal A)
ssh -i ~/.ssh/id_rsa_chameleon -N -L 16443:127.0.0.1:6443 cc@129.114.25.166

# 2) pull fresh kubeconfig from NEW cluster node1 (terminal B)
scp -i ~/.ssh/id_rsa_chameleon cc@129.114.24.249:/home/cc/admin.conf ~/.kube/proj10-new-admin.conf
chmod 600 ~/.kube/proj10-new-admin.conf

# 3) use that kubeconfig and point server to tunnel
export KUBECONFIG=~/.kube/proj10-new-admin.conf

# 4) make sure to set the current kubectl config as the remote k8s cluster
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.contexts[0].context.cluster}')
kubectl config set-cluster "$CLUSTER_NAME" --server=https://127.0.0.1:16443 --insecure-skip-tls-verify=true

# 5) verify access
kubectl get nodes
kubectl -n kube-system get deploy,svc | grep -i sealed || true

# 6) seal secrets
ansible-playbook -i ./inventory.yml ./secrets/seal_k8s_secrets.yml --ask-vault-pass
```

### 6.3 Apply sealed secrets to cluster

```bash
ansible-playbook -i ./inventory.yml ./secrets/apply_k8s_secrets.yml --ask-vault-pass
```

## 7) Deploy Namespaces, Platform, Services, Apps, Workflows

Run these playbooks in order:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ./inventory.yml ./argocd/argocd_add_platform.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/argocd_add_pipeline.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/argocd_add_staging.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/argocd_add_canary.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/argocd_add_prod.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/workflow_templates_apply.yml --ask-vault-pass
```

This applies Argo workflow templates including:

- `train-model`
- `build-container-image`
- `deploy-container-image`
- `test-staging`
- `promote-model`
- `cron-train`
- `cron-backup`

## 8) Build Required Bootstrap Images / Workflows

Build backup-tools image used by `cron-backup`:

```bash
ansible-playbook -i ./inventory.yml ./ops/build_backup_tools_image.yml
```

Optional initial image/workflow bootstrap:

```bash
ansible-playbook -i ./inventory.yml ./argocd/workflow_build_ml_services_init.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/workflow_build_training_init.yml --ask-vault-pass
ansible-playbook -i ./inventory.yml ./argocd/workflow_build_init.yml --ask-vault-pass
```

Optional: seed Argo source cache PVC from local repos (helps when GitHub network is flaky):

```bash
ansible-playbook -i ./inventory.yml ./ops/seed_argo_source_cache.yml
```

## 9) Backup and Restore

### 9.0 Mount / Remount / Unmount backup volume

Use this when moving the persistent backup volume across rebuilt clusters.

Inspect block devices:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

Mount the volume on `node1` (example device `/dev/vdb`):

```bash
sudo mkdir -p /mnt/mlops-backup
sudo mount /dev/vdb /mnt/mlops-backup
df -h | grep mlops-backup
```

Persist mount across reboot:

```bash
echo '/dev/vdb /mnt/mlops-backup ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a
```

Remount after changes to `/etc/fstab` or mount options:

```bash
sudo umount /mnt/mlops-backup
sudo mount /mnt/mlops-backup
```

Unmount before detaching volume from the node:

```bash
sudo sync
sudo umount /mnt/mlops-backup
```

Clean up stale fstab entry if this node will no longer mount that disk:

```bash
sudo sed -i.bak '\# /mnt/mlops-backup #d' /etc/fstab
```

Important:

- Do not run `mkfs` on an existing backup disk.
- Always `umount` cleanly before detaching in OpenStack.

### 9.1 Create backup

```bash
ansible-playbook -i ./inventory.yml ./ops/backup_everything.yml --ask-vault-pass
```

### 9.2 Restore data to rebuilt cluster

Use backup ID from `/mnt/mlops-backup/<backup_id>`.

```bash
ansible-playbook -i ./inventory.yml ./ops/restore_everything.yml --ask-vault-pass \
  -e "backup_id=<backup_id> restore_postgres=true restore_minio=true restore_argo_state=false restore_argocd_apps=false"
```

Why keep `restore_argo_state=false restore_argocd_apps=false` by default:

- Avoid reapplying stale workflow/app objects that can conflict with current repo state.
- Prefer repo-as-source-of-truth via `argocd_add_*` and `workflow_templates_apply`.

Enable either flag only when you intentionally want object-state replay from backup files.

## 10) Post-Deploy Ops Playbooks

Run one-shot data generator job:

```bash
ansible-playbook -i ./inventory.yml ./ops/run_data_generator_once.yml \
  -e "run_seconds=600 journeys_per_sec=2.0 gemspot_rps=0.2"
```

Run one-shot ETL metrics backfill:

```bash
ansible-playbook -i ./inventory.yml ./ops/run_etl_metrics_backfill_once.yml --ask-vault-pass
```

## 11) Verification Checklist

```bash
kubectl get nodes -o wide
kubectl get ns
kubectl -n argocd get applications
kubectl -n argo get workflowtemplates,cronworkflows
kubectl -n proj10-platform get pods
kubectl -n proj10-staging get pods
kubectl -n proj10-canary get pods
kubectl -n proj10-production get pods
```

Also validate:

- `proj10-platform` secrets exist (`postgres-credentials`, `minio-credentials`, `airflow-secrets`, `nimtable-*`).
- `argo` has `github-repo-auth` and runtime copied secrets.
- `cron-train` and `cron-backup` are present and not suspended (unless intentionally paused).

## 12) Common Failure Modes

- `Could not match supplied host pattern` during Kubespray:
  inventory group names mismatch. Use `kube_control_plane`, `kube_node`, `k8s_cluster`.
- `admin.conf not found` in post-k8s:
  cluster bootstrap did not complete correctly; verify control-plane setup first.
- ArgoCD sync says operation already in progress:
  terminate in-flight op and retry.
- Workflow/Git clone DNS failures:
  use `./ops/seed_argo_source_cache.yml` to avoid hard dependency on live GitHub access.
- `ImagePullBackOff` on public tags:
  pin to image tags that exist, or build/push into internal registry first.

## Security Notes

- This is a lab/course environment and includes permissive defaults (e.g., insecure internal registry).
- Never commit real plaintext credentials.
- Keep sealed secrets and vault files managed carefully.
