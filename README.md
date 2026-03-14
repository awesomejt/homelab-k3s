# Homelab K3s GitOps

This repository holds steady-state cluster configuration for the homelab K3s clusters.
Bootstrap-only work stays in the Ansible repository: installing K3s, installing Argo CD,
exposing Argo CD, adding the Git repository credential, and creating the initial root
application.

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

* `metallb-<env>` installs the MetalLB controller from the upstream Helm chart
* `metallb-config-<env>` applies the address pool and L2 advertisement from this repo
* `external-dns-<env>` installs ExternalDNS from the upstream Helm chart for automatic DNS record management
* `harbor-<env>` installs Harbor from the upstream Helm chart as the in-cluster container registry
* `headlamp-<env>` installs Headlamp from the upstream Helm chart and exposes it via ingress

This keeps the split clean:

* Ansible owns bootstrap and break-glass recovery
* Argo CD owns everything after bootstrap

## Bootstrap Flow

1. Provision VMs and baseline OS configuration with the Proxmox and Ansible repos.
2. Run the Ansible K3s playbook for `stage` or `prod`.
3. Ansible installs K3s, optional cert-manager, Argo CD, the Argo CD ingress, the GitOps repo credential, and the root Argo CD application.
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
* Domain filter: `taylor.lan`
* Sources: Kubernetes `Service` and `Ingress`

The `external-dns-rfc2136` secret is seeded by the Ansible K3s bootstrap role from vault values in `ansible/vars/<env>/secrets.yaml`:

* `vault_external_dns_rfc2136_tsig_keyname`
* `vault_external_dns_rfc2136_tsig_secret`
* `vault_external_dns_rfc2136_tsig_secret_alg` (optional, defaults to `hmac-sha256`)

This keeps sensitive DNS credentials out of Git while still allowing GitOps-managed application configuration.

## Harbor

Harbor is installed per environment from the upstream Helm chart and exposed through Traefik ingress.
The current chart pin is `1.18.0`, which maps to Harbor app version `2.14`.

Application manifests:

* `clusters/stage/harbor.yaml`
* `clusters/prod/harbor.yaml`

Current defaults:

* Stage hostname: `harbor-stage.taylor.lan`
* Prod hostname: `harbor-prod.taylor.lan`
* TLS source: Harbor chart auto-generated certificate
* Storage class: `local-path`
* Update strategy: `Recreate` for PVC-backed components on RWO storage

Review the ingress hostnames, storage sizing, and admin password in those manifests before first sync. If you want trusted TLS for Docker and Helm clients, replace the auto-generated certificate flow with a secret issued by your preferred CA or cert-manager.

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