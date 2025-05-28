# Homelab K3s Cluster

## Requirements/Assumptions

* SSH setup to VMs
* VMs will be up to date
* Determine a K3s token in advance

## Provisioning

* Create 4-5 Ubuntu VMs
    * k3s-server: control plane server
    * k3s-node-1: k3s worker node
    * k3s-node-2: k3s worker node
    * k3s-node-3: k3s worker node
    * k3s-admin: kubernetes admin (optional)

Keep track of the IP addresses of each VM. For example:

```
192.168.50.55 k3s-server
192.168.50.56 k3s-node-1
192.168.50.57 k3s-node-1
192.168.50.58 k3s-node-1
192.168.50.52 k3s-admin
```

This will be added to the /etc/hosts file of each VM.

## Setup K3s Server

Logon to VM that will become the K3s server.

Copy IP address contents to /etc/hosts file:

```bash
nano /etc/hosts
```

Paste contents of k3s VM IP Addresses.

Run k3s server install:

```bash
curl -sfL https://get.k3s.io | sudo K3S_TOKEN="mysecret" sh -
```

Change the token value accordingly.

Copy K3s config to user directory and change owner of file:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml .
chown jason:jason k3s.yaml  # change owner to user
```

## Kubectl on Windows

```bash
mkdir ~/bin  #if not created yet
cd ~/bin
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/windows/amd64/kubectl.exe"
```

Append .bashrc file:

```bash
alias k="kubectl"

PATH="~/bin:$PATH"
```

Copy K3s file from K3s server to local system:

```bash
scp jason@192.168.50.55:~/k3s.yaml .
```

Edit k3s.yaml file to change:

```yaml
clusters:
- cluster:
    server: https://192.168.50.55:6443
```

Change server field to point to K3s server IP or hostname on local network

Copy K3s file to local Kube config:

```bash
mkdir .kube
mv k3s.yaml .kube/config
```

## K3s Nodes

Logon to each k3s worker node.

Edit /etc/hosts file like above.

Run k3s node install:

```bash
curl -sfL https://get.k3s.io | sudo K3S_TOKEN="mysecret" K3S_URL=https://k3s-server:6443 sh
```

## Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Install argocd cli (windows):

```bash
cd ~/bin
curl -o argocd.exe -L https://github.com/argoproj/argo-cd/releases/download/v3.0.3/argocd-windows-amd64.exe
```

Port forward ArgoCD service:

Grab "admin" secret:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml

# take password and base64 decode

echo <password> | base64 --decode
```

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Use web browser to access: https://127.0.0.1:8080/ 

Browser should complain, but access it anyway.

```bash
# go into this repo folder on local system to apply application to ArgoCD
cd ~/projects/homelab-k3s
kubectl apply -f apps/example-app.yaml
kubectl port-forward svc/homelab-nginx-service -n default 9080:80
```

Open web browser: http://127.0.0.1:9080/