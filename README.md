# Homelab K3s GitOps

This repository holds steady-state cluster configuration for homelab K3s clusters.
Bootstrap-only work stays in the Ansible repository (install K3s, install Argo CD,
configure SOPS decryption key, add repository SSH credential, and create root app).

## Cluster Strategy

The repository is organized for three cluster environments:

- `clusters/dev`
- `clusters/stage`
- `clusters/prod`

Implementation is dev-first. Services are validated in `dev`, then promoted to `stage`, then `prod`.

## Core Services (Target State)

- MetalLB
- Traefik (ingress controller)
- cert-manager
- ExternalDNS
- Headlamp
- Argo Workflows
- kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
- Longhorn
- Argo CD configuration managed from GitOps

Deliberately excluded from this repo:

- Harbor
- Reposilite

Nexus on a dedicated VM is used for image and artifact registry duties.

## Repository Layout

```text
clusters/
  dev/                      # Root Argo CD app path for dev
  stage/                    # Root Argo CD app path for stage
  prod/                     # Root Argo CD app path for prod

apps/
  infrastructure/
    argocd/overlays/dev/                  # Argo CD self-managed config
    traefik-config/overlays/dev/          # K3s Traefik HelmChartConfig
    cert-manager/overlays/dev/values.yaml
    cert-manager-issuers/base/
    metallb-controller/overlays/dev/values.yaml
    metallb-config/overlays/{dev,stage,prod}/
    external-dns/overlays/dev/values.yaml
    headlamp/overlays/dev/values.yaml
    argo-workflows/overlays/dev/values.yaml
    kube-prometheus-stack/overlays/{dev,stage,prod}/values.yaml
    longhorn/overlays/dev/values.yaml
```

Helm values are split into dedicated files under `apps/infrastructure/<service>/overlays/<env>/values.yaml`.

## App Of Apps Pattern

Each `clusters/<env>` path contains:

- An `AppProject` for that environment
- Child Argo CD `Application` resources for cluster services

For dev, applications are currently ordered by sync wave:

1. `argocd-config-dev`
2. `traefik-config-dev`
3. `metallb-dev`
4. `cert-manager-dev`
5. `cert-manager-issuers-dev`
6. `metallb-config-dev`
7. `longhorn-dev`
8. `external-dns-dev`
9. `kube-prometheus-stack-dev`
10. `headlamp-dev`
11. `argo-workflows-dev`

## Bootstrap Flow

1. Provision nodes and base OS from Proxmox/Ansible.
2. Run Ansible K3s playbook for the environment.
3. Run Ansible bootstrap playbook to install Argo CD and seed repo/age keys.
4. Argo CD syncs `clusters/<env>` from this repository.
5. Child apps reconcile continuously.

## SOPS Secret Management

Runtime secrets are committed only as SOPS-encrypted manifests.

### `.sops.yaml`

`.sops.yaml` is safe to commit because it contains the age public key, not the private key.

Current rule:

```yaml
creation_rules:
  - path_regex: ^apps/.*\.yaml$
    age: age1hsjjjnyycyz2r2c4rm9zkctcm0wz9nm4pwcq0ka5pzpypvthxfaqgqn888
```

### Secrets That Must Be SOPS-Encrypted

- `external-dns-rfc2136` TSIG secret (`external-dns` namespace)
- `grafana-admin` secret (`monitoring` namespace)
- Any Argo Workflows SSO/OIDC client secrets (if enabled later)
- Any Longhorn backup target credentials (if using S3/NFS backup target)
- Any additional app credentials introduced under `apps/**`

Bootstrap-only secrets stay in Ansible Vault, not this repository:

- Argo CD repository SSH private key
- SOPS age private key

## Dev Implementation Checklist

- [x] Remove Harbor and Reposilite from cluster kustomizations
- [x] Add dev AppProject and child Applications for core services
- [x] Split dev Helm values into `apps/infrastructure/*/overlays/dev/values.yaml`
- [x] Add dev Traefik config via `HelmChartConfig`
- [x] Add dev Longhorn and Argo Workflows applications
- [x] Add dev Argo CD config application
- [ ] Encrypt and commit dev `external-dns-rfc2136` secret manifest
- [ ] Encrypt and commit dev `grafana-admin` secret manifest
- [ ] Sync dev and validate health for all core services
- [ ] Promote working manifests to `stage`
- [ ] Promote working manifests to `prod`

## Recommended Additional Homelab Services

- `reloader` for auto-restarting pods on ConfigMap/Secret changes
- `descheduler` for periodic rebalance and cleanup on small clusters
- `metrics-server` tuning (if needed) for HPA stability
- `sealed-secrets` only if you want contributor-friendly secret UX beyond SOPS
- `node-problem-detector` for better node-level alerting

## Notes

- `step-ca` is already hosted on a dedicated VM and can be integrated later as an issuer backend.
- This repo assumes Traefik as ingress controller (no NGINX ingress usage).
- Keep bootstrap and break-glass procedures in Ansible; keep steady-state in GitOps.