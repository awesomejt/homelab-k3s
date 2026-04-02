# Bootstrap Access Runbook

This guide explains how to access and validate your K3s clusters and Argo CD from your local workstation.

It covers:

- Installing kubectl on Ubuntu-based distros and Arch/CachyOS
- Copying K3s kubeconfig files from cluster nodes
- Merging configs into one local kubeconfig
- Fixing common server hostname and certificate issues
- Creating quick context switch aliases (`kdev`, `kstage`, `kprod`)
- Validating cluster and Argo CD health
- Accessing Argo CD before ingress is available

## 1. Install kubectl

### Ubuntu and Ubuntu-based distros

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
kubectl version --client
```

### Arch Linux and CachyOS

```bash
sudo pacman -Syu --needed kubectl
kubectl version --client
```

Optional helpers:

```bash
sudo pacman -S --needed k9s helm
```

## 2. Copy K3s kubeconfig files from servers

K3s writes kubeconfig to:

```text
/etc/rancher/k3s/k3s.yaml
```

Copy one kubeconfig from each environment control-plane server to your workstation:

```bash
mkdir -p ~/.kube/homelab

scp jason@<dev-server-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/homelab/dev.yaml
scp jason@<stage-server-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/homelab/stage.yaml
scp jason@<prod-server-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/homelab/prod.yaml
```

If direct scp fails due to permissions, copy as root on the server first:

```bash
ssh jason@<server-ip> 'sudo cp /etc/rancher/k3s/k3s.yaml /tmp/k3s.yaml && sudo chown jason:jason /tmp/k3s.yaml'
scp jason@<server-ip>:/tmp/k3s.yaml ~/.kube/homelab/<env>.yaml
ssh jason@<server-ip> 'rm -f /tmp/k3s.yaml'
```

## 3. Fix server endpoint per environment

Each copied file usually points to `https://127.0.0.1:6443`, which is only valid on the server itself.
Update each file to use a reachable address or DNS name for that cluster API endpoint.

```bash
# Example edits
sed -i 's#https://127.0.0.1:6443#https://<dev-api-ip-or-dns>:6443#' ~/.kube/homelab/dev.yaml
sed -i 's#https://127.0.0.1:6443#https://<stage-api-ip-or-dns>:6443#' ~/.kube/homelab/stage.yaml
sed -i 's#https://127.0.0.1:6443#https://<prod-api-ip-or-dns>:6443#' ~/.kube/homelab/prod.yaml
```

## 4. Merge kubeconfigs into one file

```bash
KUBECONFIG=~/.kube/homelab/dev.yaml:~/.kube/homelab/stage.yaml:~/.kube/homelab/prod.yaml \
  kubectl config view --flatten > ~/.kube/config

chmod 600 ~/.kube/config
kubectl config get-contexts
```

## 5. Rename contexts to dev/stage/prod

K3s kubeconfigs often use generic names. Rename to consistent context names:

```bash
# List contexts first
kubectl config get-contexts -o name

# Example renames (adjust old names to your actual context names)
kubectl config rename-context default dev
kubectl config rename-context default-1 stage
kubectl config rename-context default-2 prod
```

Set default context:

```bash
kubectl config use-context dev
```

## 6. Fix common hostname or certificate errors

### Error: x509 certificate is valid for 127.0.0.1, not <api-ip-or-dns>

Preferred fix:

- Configure K3s server with correct TLS SAN entries (VIP/DNS/IP) and restart K3s.
- Re-copy `/etc/rancher/k3s/k3s.yaml`.

Temporary workaround for testing only:

```bash
kubectl config set-cluster <cluster-name> --insecure-skip-tls-verify=true
kubectl config set-cluster <cluster-name> --certificate-authority=
```

Do not keep this in long-term production usage.

### Error: certificate signed by unknown authority

- Ensure `certificate-authority-data` is present in kubeconfig.
- Re-copy config from the server; do not strip CA data.

### Error: dial tcp ... connect: no route to host / timeout

- Confirm server field points to reachable IP/DNS and port 6443.
- Check firewall/routing between your workstation and cluster API endpoint.
- Verify k3s server is active: `sudo systemctl status k3s`.

## 7. Add context switch aliases

Add these to `~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`.

### Bash/Zsh

```bash
alias kdev='kubectl config use-context dev && kubectl config current-context'
alias kstage='kubectl config use-context stage && kubectl config current-context'
alias kprod='kubectl config use-context prod && kubectl config current-context'
alias kctx='kubectl config current-context'
```

### Fish (CachyOS default shell often uses fish)

```fish
function kdev
  kubectl config use-context dev; and kubectl config current-context
end

function kstage
  kubectl config use-context stage; and kubectl config current-context
end

function kprod
  kubectl config use-context prod; and kubectl config current-context
end

function kctx
  kubectl config current-context
end
```

Reload shell config after adding aliases/functions.

## 8. Test Kubernetes access

```bash
# Switch to each context and validate
kdev
kubectl get nodes -o wide

kstage
kubectl get nodes -o wide

kprod
kubectl get nodes -o wide
```

You should see all expected server/agent nodes in `Ready` state.

## 9. Confirm Argo CD is installed and healthy

```bash
kdev
kubectl -n argocd get pods
kubectl -n argocd get applications
kubectl -n argocd get svc argocd-server
```

Healthy baseline:

- `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, and `argocd-redis` are `Running`
- Root app exists and is syncing/synced

## 10. Access Argo CD before ingress is available

Use port-forward:

```bash
kdev
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Open:

```text
https://localhost:8080
```

Get initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
```

Optional CLI login:

```bash
argocd login localhost:8080 --username admin --grpc-web --insecure
```

After first login, rotate admin password.

## 11. Useful diagnostics

### Cluster diagnostics

```bash
kubectl get nodes -o wide
kubectl get ns
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
kubectl top nodes
kubectl top pods -A
```

### Argo CD diagnostics

```bash
kubectl -n argocd get pods -o wide
kubectl -n argocd describe pods -l app.kubernetes.io/name=argocd-repo-server
kubectl -n argocd logs deploy/argocd-repo-server --tail=200
kubectl -n argocd logs deploy/argocd-application-controller --tail=200
kubectl -n argocd get applications -o wide
```

### DNS and ingress diagnostics

```bash
kubectl -n external-dns logs deploy/external-dns --tail=200
kubectl -n argocd get ingress
kubectl -n argocd describe ingress argocd-server-ingress
```

## 12. Argo CD UI Troubleshooting (Dev First)

If you have not bootstrapped stage/prod yet, focus on the dev context only.

```bash
kubectl config use-context dev
kubectl config current-context
```

Validate Argo CD namespace and config map:

```bash
kubectl -n argocd get ns argocd
kubectl -n argocd get configmap argocd-cm
kubectl -n argocd get pods
```

If `argocd-cm` is missing, force a sync of `argocd-config-dev` in Argo CD and verify:

```bash
kubectl -n argocd get application argocd-config-dev -o wide
kubectl -n argocd get configmap argocd-cm
```

Restart Argo CD deployments from CLI:

```bash
kubectl -n argocd rollout restart deploy/argocd-server
kubectl -n argocd rollout restart deploy/argocd-repo-server
kubectl -n argocd rollout restart deploy/argocd-application-controller
kubectl -n argocd rollout restart deploy/argocd-redis

kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd rollout status deploy/argocd-repo-server
kubectl -n argocd rollout status deploy/argocd-application-controller
kubectl -n argocd rollout status deploy/argocd-redis
```

Check Argo CD server logs for UI/config loading failures:

```bash
kubectl -n argocd logs deploy/argocd-server --tail=200
```

Ingress and DNS verification:

```bash
kubectl -n argocd get ingress argocd-server-ingress
kubectl -n argocd describe ingress argocd-server-ingress
kubectl -n external-dns logs deploy/external-dns --tail=200 | grep argocd.dev.lab
```

Fallback access path (if ingress is not healthy):

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Open `https://localhost:8080` and test login.

## 13. Suggested daily workflow

```bash
# pick environment
kdev

# quick health pass
kubectl get nodes
kubectl -n argocd get applications
kubectl -n argocd get pods
```

Repeat for `kstage` and `kprod` before promoting changes.
