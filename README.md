# Homelab K3s GitOps

This repository holds steady-state cluster configuration for the homelab K3s clusters.
Bootstrap-only work stays in the Ansible repository: installing K3s, installing Argo CD,
configuring KSOPS/SOPS decryption, adding the Git repository credential, and creating the
initial root application.

## Layout

```text
clusters/
    prod/                  # Root app path for the prod cluster
    stage/                 # Root app path for the stage cluster
apps/
    infrastructure/
        metallb-config/      # MetalLB address pool and L2Advertisement manifests
example/                 # Sample LoadBalancer workload for smoke testing
```

## App Of Apps Pattern

Each cluster environment has its own root path under `clusters/<env>`.

That root path contains:

* An `AppProject` for that cluster
* Child `Application` resources for cluster services

Current child applications:

* `cert-manager-issuers-<env>` applies shared cert-manager `ClusterIssuer` resources from this repo
* `metallb-<env>` installs the MetalLB controller from the upstream Helm chart
* `metallb-config-<env>` applies the address pool and L2 advertisement from this repo
* `external-dns-<env>` installs ExternalDNS from the upstream Helm chart for automatic DNS record management
* `harbor-<env>` installs Harbor from the upstream Helm chart as the in-cluster container registry
* `headlamp-<env>` installs Headlamp from the upstream Helm chart and exposes it via ingress
* `reposilite-<env>` installs Reposilite from custom manifests as the in-cluster Maven artifact repository
* `kube-prometheus-stack-<env>` installs Prometheus, Grafana, and Alertmanager from the upstream Helm chart for monitoring

This keeps the split clean:

* Ansible owns bootstrap and break-glass recovery
* Argo CD owns everything after bootstrap

## Bootstrap Flow

1. Provision VMs and baseline OS configuration with the Proxmox and Ansible repos.
2. Run the Ansible K3s playbook for `stage` or `prod`.
3. Ansible installs K3s first, then runs a separate bootstrap playbook that installs Argo CD,
   configures KSOPS/SOPS decryption with an age key, adds the GitOps repo credential, and creates
   the root Argo CD application.
4. Argo CD syncs `clusters/<env>` from this repo.
5. MetalLB is installed and configured from Git.

## SSH Repository Access

The Argo CD bootstrap in the Ansible repo expects this repository to be reachable over SSH.

Generate a dedicated read-only deploy key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd-homelab-k3s -C "argocd-homelab-k3s" -N ""
```

Add the public key as a read-only deploy key on the GitHub repository.

Store the private key in the matching Ansible environment vault file:

```yaml
vault_argocd_gitops_repo_ssh_private_key: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

The GitOps repo URL used by the bootstrap flow is:

```text
git@github.com:awesomejt/homelab-k3s.git
```

## SOPS Secret Management

Runtime application secrets should be committed in this repository only as SOPS-encrypted
manifests. The age private key is stored in Ansible Vault and seeded during bootstrap as a
Kubernetes secret in the `argocd` namespace.

### Setup: Configure `.sops.yaml`

Create a `.sops.yaml` file in the repository root to define the encryption key:

```yaml
creation_rules:
  - path_regex: ^apps/.*\.yaml$
    age: age1<your-public-key-here>
```

The public key corresponds to the `vault_sops_age_key` in `ansible/vars/<env>/secrets.yaml`.

To find your public key if you only have the private key:

```bash
# Extract public key from age private key
grep "# public key:" ~/.config/sops/age/keys.txt | sed 's/# public key: //'
```

### Creating Encrypted Secrets

Example: Create a secret for ExternalDNS RFC2136 TSIG credentials:

```bash
# Create plaintext manifest
cat > apps/infrastructure/external-dns/base/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: external-dns-rfc2136
  namespace: external-dns
type: Opaque
stringData:
  tsig-secret: "your-tsig-key-here"
  tsig-keyname: "externaldns.lab"
EOF

# Encrypt it with SOPS
sops -e -i apps/infrastructure/external-dns/base/secret.yaml

# Commit the encrypted version
git add apps/infrastructure/external-dns/base/secret.yaml
git commit -m "feat: add external-dns TSIG secret (encrypted)"
git push
```

Once pushed, Argo CD will automatically decrypt the secret using the age key seeded in the `argocd`
namespace during bootstrap. You can verify decryption works by checking the secret in the cluster:

```bash
# After Argo CD syncs
k3s kubectl -n external-dns get secret external-dns-rfc2136 -o jsonpath='{.data.tsig-secret}' | base64 -d
```

### Common Encrypted Secrets

Example paths for application secrets:

```
apps/infrastructure/externa-dns/base/secret.yaml         # ExternalDNS TSIG
apps/infrastructure/harbor/base/secret.yaml               # Harbor admin credentials
apps/infrastructure/reposilite/base/secret.yaml           # Reposilite credentials
apps/infrastructure/kube-prometheus-stack/base/secret.yaml # Grafana admin password
```

Example: Grafana admin password:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin
  namespace: monitoring
type: Opaque
stringData:
  admin-password: "your-strong-password"
```

Reference from your Helm values:

```yaml
# In values overlay or Helm release
adminPassword: <ENCRYPTED-SECRET-GOES-HERE>  # OR reference via ExternalSecret
# Better: use ExternalSecret + secret reference
```

### Recommended Pattern

* Keep encrypted secret manifests under `apps/**` alongside the workloads that consume them.
* Reference those secrets from your Helm values or manifests.
* Keep bootstrap-only credentials (like Argo CD repo SSH key and age private key) in Ansible Vault.

This keeps plaintext secrets out of GitHub while preserving GitOps reconciliation.

## MetalLB Address Pools

MetalLB configuration lives here:

* `apps/infrastructure/metallb-config/overlays/stage`
* `apps/infrastructure/metallb-config/overlays/prod`

Default pools are set to the high end of the `192.168.50.0/24` network:

* Stage: `192.168.50.240-192.168.50.244`
* Prod: `192.168.50.245-192.168.50.250`

Adjust those ranges before first sync if those addresses are already reserved elsewhere in the homelab.

## ExternalDNS

ExternalDNS is installed per environment from the upstream Helm chart and configured for RFC2136.

Application manifests:

* `clusters/stage/external-dns.yaml`
* `clusters/prod/external-dns.yaml`

Current defaults:

* DNS provider: RFC2136 (`provider.name: rfc2136`)
* RFC2136 server: `192.168.50.53:53` (Technitium DNS)
* RFC2136 auth: TSIG values read from Kubernetes secret `external-dns-rfc2136`
* Domain filters: `stage.lab` (stage) and `prod.lab` (prod)
* Sources: Kubernetes `Service` and `Ingress`

The `external-dns-rfc2136` secret should be managed in this repository as a SOPS-encrypted
Kubernetes Secret manifest and applied by Argo CD.

## Cert-Manager Issuers

Cluster-wide cert-manager issuer resources are managed in GitOps so certificate naming stays consistent across environments.

Application manifests:

* `clusters/stage/cert-manager-issuers.yaml`
* `clusters/prod/cert-manager-issuers.yaml`

Current default:

* ClusterIssuer name: `letsencrypt-lab`
* Issuer type: `selfSigned`

All ingress annotations in this repo reference `cert-manager.io/cluster-issuer: letsencrypt-lab`.

## Harbor

Harbor is installed per environment from the upstream Helm chart and exposed through Traefik ingress.
The current chart pin is `1.18.0`, which maps to Harbor app version `2.14`.

Application manifests:

* `clusters/stage/harbor.yaml`
* `clusters/prod/harbor.yaml`

Current defaults:

* Stage hostname: `harbor.stage.lab`
* Prod hostname: `harbor.prod.lab`
* TLS source: Harbor chart auto-generated certificate
* Storage class: `local-path`
* Update strategy: `Recreate` for PVC-backed components on RWO storage

Review the ingress hostnames, storage sizing, and admin password in those manifests before first sync. If you want trusted TLS for Docker and Helm clients, replace the auto-generated certificate flow with a secret issued by your preferred CA or cert-manager.

## Reposilite

Reposilite is installed per environment from custom Kubernetes manifests and exposed through Traefik ingress.

Application manifests:

* `clusters/stage/reposilite.yaml`
* `clusters/prod/reposilite.yaml`

Current defaults:

* Stage hostname: `artifacts.stage.lab`
* Prod hostname: `artifacts.prod.lab`
* TLS source: cert-manager auto-generated certificate
* Storage class: `local-path` (20Gi PVC)
* Admin credentials: Managed as SOPS-encrypted Kubernetes Secret manifests in this repo

The `reposilite-admin` secret should be managed as a SOPS-encrypted Kubernetes Secret manifest
in this repository.

## Kube Prometheus Stack

Prometheus, Grafana, and Alertmanager are installed per environment from the upstream Helm chart and exposed through Traefik ingress.

Application manifests:

* `clusters/stage/kube-prometheus-stack.yaml`
* `clusters/prod/kube-prometheus-stack.yaml`

Current defaults:

* Stage hostnames: `grafana.stage.lab`, `prometheus.stage.lab`, `alertmanager.stage.lab`
* Prod hostnames: `grafana.prod.lab`, `prometheus.prod.lab`, `alertmanager.prod.lab`
* TLS source: cert-manager auto-generated certificates
* Grafana admin credentials: Managed as SOPS-encrypted Kubernetes Secret manifests in this repo

The `grafana-admin` secret should be managed as a SOPS-encrypted Kubernetes Secret manifest in
this repository. Grafana is automatically configured with Prometheus as a data source.

## Adding More Apps

To add another cluster service or workload:

1. Add the application manifests under `apps/`
2. Add a child Argo CD `Application` manifest under `clusters/<env>`
3. Commit and push the change
4. Let Argo CD reconcile it

Keep anything needed to create the first working Argo CD instance in the Ansible repo, not here.

## Smoke Test

The `example/` directory contains a simple `LoadBalancer` nginx workload you can point an Argo CD application at after MetalLB is healthy.
It is intentionally not part of the default cluster bootstrap.