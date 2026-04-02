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
