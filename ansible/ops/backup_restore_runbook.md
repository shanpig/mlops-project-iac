# Full Rebuild + Restore Runbook (After Lease/Reservation Expiry)

This runbook covers end-to-end recovery: reserve new resources, rebuild cluster/services, reattach backup volume, restore data, and validate everything.

## 0) Before Lease Ends (Create Backup)

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/ansible
ansible-playbook -i inventory.yml ops/backup_everything.yml
```

Backup output:
- `/mnt/mlops-backup/<timestamp>/...`

Keep the `<timestamp>` value (`backup_id`) for restore.

## 1) Reserve New Resources

1. Create/activate new Chameleon lease and reservation.
2. Ensure flavor sizes match your cluster plan (control + workers, plus any large ML node).
3. Ensure security groups/floating IP policy are ready.

## 2) Provision Infrastructure

From your Terraform directory (where your OpenStack config lives):

```bash
terraform init
terraform plan
terraform apply
```

Then confirm all nodes are reachable:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/ansible
ansible -i inventory.yml all -m ping -u cc
```

## 3) Configure Cluster (Kubernetes)

Run your standard cluster bootstrap flow:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/ansible
ansible-playbook -i inventory.yml pre_k8s/pre_k8s_configure.yml -b -u cc
ansible-playbook -i inventory.yml k8s/kubespray/cluster.yml -b -u cc
ansible-playbook -i inventory.yml post_k8s/post_k8s_configure.yml -b -u cc
```

If you maintain dedicated ML nodes:

```bash
ansible-playbook -i inventory.yml ops/dedicate_ml_node.yml -b -u cc
```

## 4) Reattach and Mount Backup Volume

Attach volume `mlops-devops-proj10` to `node1`, then mount:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
sudo mkdir -p /mnt/mlops-backup
sudo mount /dev/vdb /mnt/mlops-backup
echo '/dev/vdb /mnt/mlops-backup ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
df -h | grep mlops-backup
```

Notes:
- Do **not** run `mkfs` if this volume contains your backups already.
- If new empty volume, format once before mount.

## 5) Recreate Platform and App Control Plane

Apply secrets and ArgoCD apps from IaC source of truth:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/ansible
ansible-playbook -i inventory.yml secrets/prepare_k8s_secrets.yml --ask-vault-pass
ansible-playbook -i inventory.yml secrets/seal_k8s_secrets.yml
ansible-playbook -i inventory.yml secrets/apply_k8s_secrets.yml

ansible-playbook -i inventory.yml argocd/argocd_add_platform.yml --ask-vault-pass
ansible-playbook -i inventory.yml argocd/argocd_add_pipeline.yml --ask-vault-pass
ansible-playbook -i inventory.yml argocd/argocd_add_staging.yml --ask-vault-pass
ansible-playbook -i inventory.yml argocd/argocd_add_canary.yml --ask-vault-pass
ansible-playbook -i inventory.yml argocd/argocd_add_prod.yml --ask-vault-pass
```

Apply workflow templates:

```bash
ansible-playbook -i inventory.yml argocd/workflow_templates_apply.yml --ask-vault-pass
```

## 6) Restore Data from Backup Volume

Pick your backup folder:
- Example `backup_id=20260420-194500`

Restore data first:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/ansible
ansible-playbook -i inventory.yml ops/restore_everything.yml \
  -e "backup_id=20260420-194500 restore_postgres=true restore_minio=true restore_argo_state=false restore_argocd_apps=false"
```

Optionally reapply backed up Argo/ArgoCD objects:

```bash
ansible-playbook -i inventory.yml ops/restore_everything.yml \
  -e "backup_id=20260420-194500 restore_postgres=false restore_minio=false restore_argo_state=true restore_argocd_apps=true"
```

## 7) Bring Services Fully Online

Wait and verify:

```bash
kubectl get pods -A
kubectl -n proj10-platform get pods
kubectl -n proj10-staging get pods
kubectl -n proj10-canary get pods
kubectl -n proj10-production get pods
```

Check key endpoints:
- frontend/backend in each env
- `ml-api`, `ml-serving`, `stream-consumer`, `redpanda`, `redis`, `minio`, `mlflow`

## 8) Post-Restore Validation Checklist

1. PostgreSQL row counts match expected snapshots (restore playbook prints counts).
2. MinIO bucket `agent-datalake` contains expected prefixes (`iceberg/`, `training_sets/`, `snapshots/`, `metadata/`).
3. MLflow model versions/aliases exist and match expected promotion state.
4. Argo workflow templates present in `argo`.
5. ArgoCD apps synced and healthy.

## 9) Pipeline Smoke Tests

Run:
- `train-model` (or manual retrain trigger)
- `build-container-image` with known model version
- `test-staging` and confirm promotion flow behavior

## 10) Operational Notes

- Treat backed-up secret manifests as sensitive.
- Keep at least one off-volume copy for critical backups.
- After major restore, run one full ETL + retrain/test cycle to confirm end-to-end integrity.
