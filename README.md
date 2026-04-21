# proj10 IaC repository

Infrastructure-as-code for the proj10 MLOps course project. This repo provisions compute + networking, bootstraps a Kubernetes cluster, and deploys the platform/app components plus Argo Workflows templates.

## Repo layout

- `tf/kvm/`: Terraform (OpenStack) to create a 3-node cluster network + instances + a floating IP.
- `ansible/pre_k8s/`: Node prep (e.g., disable firewalld, configure Docker registry/mirror).
- `ansible/k8s/kubespray/`: Kubespray for Kubernetes installation/upgrade/reset.
- `ansible/post_k8s/`: Post-install setup (including Argo CLI and Argo Workflows/Events).
- `ansible/argocd/`: ArgoCD automation (add apps for platform/envs, apply Argo WorkflowTemplates).
- `k8s/`: Kubernetes manifestss per environment (`platform`, `staging`, `production`, `canary`).
- `workflows/`: Argo WorkflowTemplates for build/deploy/train/test/promote flows.

## Prereqs

- Terraform (v1.14.4)
- Ansible (`ansible-core==2.16.9` and `ansible==9.8.0`, for Kubespray 2.26.0)
- OpenStack credentials configured for Terraform (`clouds.yaml`)
- SSH access to the provisioned nodes (keys and correctly configured `ansible.cfg`)

## Notes / safety

- This repo is designed for a course/lab environment. Some defaults are intentionally permissive (e.g., insecure Docker registry config, secrets printed in Ansible outputs).
- Never commit real credentials/secrets.

## Using Different Reservations for Cluster Nodes

Your Terraform layout supports using different flavors/reservations per node. Here’s how to set this up:

### 1. Obtain Reservation IDs

Get the **reservation UUIDs** (not lease IDs) for your two leases:

```bash
openstack lease show mlops-restart-proj10 -f json | jq
openstack lease show mlops-restart-proj10-large -f json | jq
```

Look under each lease’s `reservations` block (`flavor:instance`) to find the reservation UUIDs.

### 2. Configure `secrets.auto.tfvars`

Set variables so that node1–3 use the `large` reservation and node4 uses the `xlarge` reservation. Edit your `secrets.auto.tfvars` file as follows:

```hcl
reservation = "<RESERVATION_ID_FOR_3x_m1.large>"

node_flavor_id_overrides = {
  node4 = "<RESERVATION_ID_FOR_1x_m1.xlarge>"
}
```

### 3. Apply Terraform

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac/tf/kvm
terraform init
terraform plan
terraform apply
```

### 4. (Re)Provision Your Cluster

#### If rebuilding the cluster from scratch:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ansible/inventory.yml ansible/pre_k8s/pre_k8s_configure.yml -b -u cc
ansible-playbook -i ansible/k8s/inventory/mycluster/hosts.yaml ansible/k8s/kubespray/cluster.yml -b -u cc
ansible-playbook -i ansible/inventory.yml ansible/post_k8s/post_k8s_configure.yml -b -u cc
```

#### If only adding node4 to an existing cluster:

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ansible/inventory.yml ansible/pre_k8s/pre_k8s_configure.yml -b -u cc --limit node4
ansible-playbook -i ansible/k8s/inventory/mycluster/hosts.yaml ansible/k8s/kubespray/scale.yml -b -u cc --limit node4
```

### 5. Restore Platform/Data

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i ansible/inventory.yml ansible/ops/restore_everything.yml --ask-vault-pass -e "backup_id=<your_backup_id>"
ansible-playbook -i ansible/inventory.yml ansible/argocd/argocd_add_platform.yml --ask-vault-pass
ansible-playbook -i ansible/inventory.yml ansible/argocd/argocd_add_pipeline.yml --ask-vault-pass
ansible-playbook -i ansible/inventory.yml ansible/argocd/workflow_templates_apply.yml --ask-vault-pass
```

---

#### :warning: Important Fix Before Running Kubespray

In your `hosts.yaml` inventory, `ansible_host` is currently set to `node1`, `node2`, ... (using hostnames). **DNS resolution may fail!**  
Set these to IP addresses (`192.168.1.11`, etc.) instead to avoid errors like “Could not resolve hostname node4”.

If you’d like, I can provide a patch for the inventory file.
