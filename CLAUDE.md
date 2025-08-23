# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes-native homelab running on K3s with GitOps deployment using FluxCD and Kustomize. The repository follows Infrastructure as Code (IaC) principles with all configurations managed through Git.

## Architecture

### GitOps Flow
- **FluxCD** continuously monitors the `main` branch and automatically deploys changes
- **Staging environment** is defined in `./clusters/staging/`
- Changes to manifests in `apps/` or `infrastructure/` directories trigger automatic deployments
- All deployments use Kustomize overlays: base configurations + environment-specific patches

### Directory Structure
- `apps/`: Application deployments (base + staging overlays)
- `infrastructure/`: Core infrastructure components (controllers + configs)
- `clusters/`: FluxCD cluster configuration and kustomizations
- Each app/component has its own namespace and directory

## Essential Commands

### Secret Management (SOPS)
```bash
# Encrypt a secret file in-place
sops -e -i apps/staging/appname/secret.yaml

# Decrypt for viewing/editing (will auto-encrypt on save)
sops apps/staging/appname/secret.yaml

# Create new encrypted secret from template
kubectl create secret generic my-secret --from-literal=key=value --dry-run=client -o yaml | sops -e --input-type=yaml --output-type=yaml /dev/stdin > secret.yaml
```

### FluxCD Operations
```bash
# Force reconciliation of apps
flux reconcile kustomization apps --with-source

# Force reconciliation of infrastructure
flux reconcile kustomization infrastructure-controllers --with-source
flux reconcile kustomization infrastructure-configs --with-source

# Check FluxCD status
flux get kustomizations
flux get helmreleases -A

# Suspend/resume deployments
flux suspend kustomization apps
flux resume kustomization apps
```

### Deployment Testing
```bash
# Test kustomize build locally
kubectl kustomize apps/staging/appname/

# Dry-run apply
kubectl apply -k apps/staging/appname/ --dry-run=client

# View generated manifests
kustomize build apps/staging/appname/
```

### GPU Workload Verification
```bash
# Check GPU availability
kubectl describe nodes | grep nvidia.com/gpu

# Verify GPU pods
kubectl get pods -A -o wide | grep -E "(ollama|jellyfin)"
```

## Key Patterns

### Adding New Applications
1. Create base configuration in `apps/base/appname/`
2. Create staging overlay in `apps/staging/appname/` with:
   - `kustomization.yaml` referencing the base
   - Environment-specific patches (ingress, resources, etc.)
   - Encrypted secrets using SOPS
3. Add namespace creation if needed
4. FluxCD will auto-deploy after merge to main

### Working with Secrets
- All secrets must be encrypted with SOPS before committing
- Age recipient: `age1sy97xhs7my3793xjeyggvam25qhdv63f05h3f3ftevqfkjsh7cpqapg6f2`
- SOPS encrypts only `data` and `stringData` fields in Secrets
- FluxCD automatically decrypts during deployment using `sops-age` secret

### Kustomize Overlay Structure
```yaml
# apps/staging/appname/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: appname
resources:
  - ../../base/appname
  - ingress.yaml  # Environment-specific
  - secret.yaml   # Encrypted with SOPS
```

### GPU-Accelerated Workloads
- Use `nvidia` runtimeClassName for GPU containers
- Add node selector: `feature.node.kubernetes.io/gpu: "true"`
- Currently used by: Ollama (AI/ML) and Jellyfin (transcoding)

## Technology Stack

- **Orchestration**: K3s with NVIDIA GPU support
- **GitOps**: FluxCD v2 with Kustomize
- **Ingress**: Traefik with SSL/TLS
- **Storage**: Longhorn (distributed), local-path, NFS/SMB
- **Monitoring**: kube-prometheus-stack (Prometheus + Grafana)
- **Secrets**: SOPS with age encryption
- **Updates**: Renovate for automated dependency management

## Important Considerations

- Never commit unencrypted secrets
- All changes should go through Git (no direct kubectl apply in production)
- FluxCD reconciles every 10 minutes by default
- Storage classes: `longhorn` for replicated, `local-path` for node-local
- External access via Cloudflare Tunnels and Tailscale
- Check Renovate PRs regularly for security updates