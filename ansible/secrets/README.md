# Kubernetes Secrets Workflow (Local + Remote)

This folder contains two playbooks:

- `prepare_k8s_secrets.yml` (run on local laptop)
- `seal_k8s_secrets.yml` (run on local laptop)
- `apply_k8s_secrets.yml` (run on remote node1)

## 1) Local: Install `kubeseal`

### macOS

```bash
brew install kubeseal
```

### Linux

```bash
KUBESEAL_VERSION='0.27.3'
curl -sSL -o /tmp/kubeseal.tar.gz "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf /tmp/kubeseal.tar.gz -C /tmp kubeseal
sudo install -m 0755 /tmp/kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

## 2) Remote (node1): Install Sealed Secrets controller in cluster

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install sealed-secrets bitnami/sealed-secrets \
  -n kube-system \
  --create-namespace \
  --set-string fullnameOverride=sealed-secrets-controller
kubectl -n kube-system rollout status deploy/sealed-secrets-controller
```

## 3) Local: Generate plain secret manifests from `.env`

```bash
ansible-playbook ansible/secrets/prepare_k8s_secrets.yml
```

Generated files are written to:

- `k8s/secrets/plain/*.yaml`

This directory is git-ignored by default.

## 4) Local: connect to remote cluster API (for cert fetch)

`kubeseal --fetch-cert` must talk to the remote cluster's Sealed Secrets controller.
If your local `kubectl` context is not the remote cluster, create an SSH API tunnel and use a temporary kubeconfig.

### 4.1 Create API tunnel from local laptop

Keep this terminal open:

```bash
ssh -N -L 16443:127.0.0.1:6443 cc@<FLOATING_IP> -i ~/.ssh/id_rsa_chameleon
```

If you use SSH config alias:

```bash
ssh -N -L 6443:127.0.0.1:6443 node1
```

### 4.2 Build temporary local kubeconfig for tunneled API

In another local terminal:

```bash
scp cc@<FLOATING_IP>:/etc/kubernetes/admin.conf /tmp/proj10-admin.conf
sed -i.bak 's#server: https://127.0.0.1:6443#server: https://127.0.0.1:6443#g' /tmp/proj10-admin.conf
rm -f /tmp/proj10-admin.conf.bak
export KUBECONFIG=/tmp/proj10-admin.conf
kubectl get ns
```

If your `admin.conf` server is `https://192.168.1.11:6443`, replace it with `https://127.0.0.1:6443`:

```bash
sed -i.bak 's#server: https://192.168.1.11:6443#server: https://127.0.0.1:6443#g' /tmp/proj10-admin.conf
rm -f /tmp/proj10-admin.conf.bak
```

## 5) Local: Seal generated manifests

Use the sealing playbook (recommended):

```bash
ansible-playbook ansible/secrets/seal_k8s_secrets.yml
```

This playbook:

- Fetches the Sealed Secrets controller cert from your current `kubectl` context
- Seals all files from `k8s/secrets/plain/*.yaml`
- Writes canonical outputs to `k8s/secrets/*-sealedsecret.yaml`
- Removes legacy duplicate names like `*-secret-sealedsecret.yaml`

Commit and push only `k8s/secrets/*-sealedsecret.yaml`.

## 6) Remote: Apply sealed secrets

```bash
ansible-playbook -i ansible/inventory.yml ansible/secrets/apply_k8s_secrets.yml
```
