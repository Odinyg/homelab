# Homelab

K3s cluster managed using Kustomize for GitOps deployment of self-hosted applications

![K3s](https://img.shields.io/badge/k3s-v1.28+-blue?style=flat-square&logo=kubernetes) ![Kustomize](https://img.shields.io/badge/kustomize-v5.0+-green?style=flat-square) ![SOPS](https://img.shields.io/badge/SOPS-encrypted-red?style=flat-square)
![FluxCD](https://img.shields.io/badge/FluxCD-v2.0+-5468FF?style=flat-square&logo=flux)

Welcome to my homelab! This repository contains the complete Infrastructure-as-Code (IaC) for my Kubernetes homelab running on K3s. The cluster hosts various self-hosted applications for media streaming, productivity, monitoring, and AI workloads, all managed through GitOps principles using Kustomize overlays.

## 🚀 Features

- **K3s Kubernetes cluster** with NVIDIA GPU support for AI inference
- **GitOps deployment** using Kustomize and FluxCD for automated continuous delivery
- **Automated dependency** management with Renovate creating pull requests for updates
- **Secret management** with SOPS and age encryption
- **External access** via Cloudflare Tunnels and Tailscale
- **Monitoring** with Prometheus and Grafana
- **Multi-storage support** including local, NFS, and Longhorn distributed storage
- **SSL/TLS termination** with Traefik ingress controller

## 🍇 Cluster

### Infrastructure Automation

The homelab uses a **GitOps approach** with FluxCD and Kustomize for automated deployment and configuration management. FluxCD continuously monitors the Git repository and automatically applies changes to the cluster, ensuring the desired state is always maintained.

### GitOps

- **FluxCD** - Automated GitOps continuous delivery and reconciliation that watches the repository for changes and automatically deploys updates
- **Renovate** - Automated dependency updates via pull requests for container images, Helm charts, etc.
- **Kustomize overlays** - Environment-specific configurations with base/staging structure
- **SOPS encryption** - Secure secret management with age keys integrated into GitOps workflows
- **Automated reconciliation** - Ensures cluster state matches Git repository at all times

### Directories

This Git repository contains the following top level directories:

```
📁 apps/                    # Applications deployed into the cluster
├─📁 base/                  # Base application configurations  
└─📁 staging/               # Environment-specific overlays
📁 infrastructure/          # Infrastructure components and controllers
├─📁 controllers/           # Cluster infrastructure (monitoring, storage, etc.)
└─📁 configs/               # Configuration overlays and secrets
```

## 🖥️ Tech Stack

### Infrastructure

|Logo|Name|Description|
|:----|:----|:--------|
|[<img width="32" src="https://raw.githubusercontent.com/cncf/artwork/main/projects/kubernetes/icon/color/kubernetes-icon-color.svg">](https://kubernetes.io) | [K3s](https://k3s.io/) | Lightweight Kubernetes distribution |
|[<img width="32" src="https://raw.githubusercontent.com/cncf/artwork/main/projects/kubernetes/icon/color/kubernetes-icon-color.svg">](https://kustomize.io/) | [Kustomize](https://kustomize.io/) | Kubernetes native configuration management |
|[<img width="32" src="https://avatars.githubusercontent.com/u/129185620?s=48&v=4">](https://github.com/mozilla/sops) | [SOPS](https://github.com/mozilla/sops) | Secrets management with age encryption |
|[<img width="32" src="https://raw.githubusercontent.com/traefik/traefik/master/docs/content/assets/img/traefik.logo.png">](https://doc.traefik.io/traefik/) | [Traefik](https://doc.traefik.io/traefik/) | Modern HTTP reverse proxy and load balancer |
|[<img width="32" src="https://longhorn.io/img/logos/longhorn-horizontal-color.png">](https://longhorn.io/docs/) | [Longhorn](https://longhorn.io/docs/) | Cloud native distributed block storage |
|[<img width="32" src="https://github.com/cncf/artwork/blob/aea0dcfe090b8f36d7ae1eb3d5fbe95cc77380d3/projects/prometheus/icon/color/prometheus-icon-color.png?raw=true">](https://prometheus.io) | [Prometheus](https://prometheus.io/docs/) | Systems monitoring and alerting toolkit |
|[<img width="32" src="https://grafana.com/static/img/menu/grafana2.svg">](https://grafana.com/docs/) | [Grafana](https://grafana.com/docs/) | Operational dashboards and visualization |
|[<img width="32" src="https://github.com/jetstack/cert-manager/raw/master/logo/logo.png">](https://cert-manager.io) | [cert-manager](https://cert-manager.io/docs/) | Cloud native certificate management |
|[![](https://avatars.githubusercontent.com/u/60239468?s=32&v=4)](https://metallb.org) | [MetalLB](https://metallb.universe.tf/) | Bare metal load balancer for HA services |
|[<img width="32" src="https://raw.githubusercontent.com/cncf/artwork/main/projects/helm/icon/color/helm-icon-color.png">](https://helm.sh) | [Helm](https://helm.sh/docs/) | The package manager for Kubernetes |
|[<img width="32" src="https://camo.githubusercontent.com/fdffb57ca7bf0ba2900bab738df7bf002dee35f15e55f2029a97de1d2bdc1e07/68747470733a2f2f7777772e70726f786d6f782e636f6d2f696d616765732f70726f786d6f782f50726f786d6f782d6c6f676f2d737461636b65642d38343070782e706e67">](https://pve.proxmox.com/wiki/Main_Page) | [Proxmox VE](https://pve.proxmox.com/wiki/Main_Page) | 3-node HA cluster with Ceph storage |
|[<img width="32" src="https://ceph.io/assets/bitmaps/Ceph_Logo_Stacked_RGB_120411_fa.png">](https://docs.ceph.com/) | [Ceph](https://docs.ceph.com/) | Distributed storage across Proxmox cluster |
|[<img width="32" src="https://user-images.githubusercontent.com/7613738/113836934-a1764e00-978d-11eb-8e19-a087c5c1f99b.png">](https://www.truenas.com/docs/scale/) | [TrueNAS Scale](https://www.truenas.com/docs/scale/) | N100 NAS with ZFS storage and application hosting |
|[<img width="32" src="https://cdn.prod.website-files.com/622b70d8906c7ab0c03f77f8/63b40a92093c6b2f3767e4e6_tMCv8T-y_400x400.webp">](https://help.ui.com/) | [UniFi Network](https://help.ui.com/) | Enterprise networking with UDM Ultra, 16-port switch, and APs |


### Applications (by namespace)

#### Media & Entertainment

| Icon | Application | Category | Description | Status |
|------|-------------|----------|-------------|---------|
| 🎬 | Jellyfin | Media Server | Self-hosted media streaming with GPU transcoding | ✅ Deployed |
| 📚 | Audiobookshelf | Audio Books | Self-hosted audiobook and podcast server | ✅ Deployed |

#### Productivity & Tools  

| Icon | Application | Category | Description | Status |
|------|-------------|----------|-------------|---------|
| 🔖 | Linkding | Bookmark Manager | Minimal bookmark management | ✅ Deployed |
| 🤖 | Ollama + Open WebUI | AI/LLM | Local large language model deployment | ✅ Deployed |

#### Gaming & Streaming

| Icon | Application | Category | Description | Status |
|------|-------------|----------|-------------|---------|
| 🎮 | Steam Headless | Cloud Gaming | Steam with Sunshine streaming server | 📦 Archived |

#### Infrastructure & Monitoring

| Icon | Application | Category | Description | Status |
|------|-------------|----------|-------------|---------|
| 📊 | Grafana | Dashboard | Operational dashboards and monitoring | ✅ Deployed |
| 🔐 | Vault | Secrets Management | HashiCorp Vault for secret management | 🚧 Testing | 
| 💾 | Longhorn | Storage Management | Distributed storage management UI | ✅ Deployed |
| 🔄 | Renovate | Automation | Automated dependency updates | ✅ Deployed |

## 🔧 Hardware Requirements

### Current Hardware
- **Proxmox Cluster**: 3-node HA cluster with Ceph distributed storage
  - **Main Station PC**: Primary node with NVIDIA RTX 3090 for GPU workloads
  - **XPS 15**: Laptop node with 5Gb WizDPI networking
  - **Razer 15**: Laptop node with 5Gb WizDPI networking
- **TrueNAS Scale**: N100-based NAS with 4x5Gb networking
  - **Services**: Jellyfin (main instance), PostgreSQL, Redis
  - **Planned**: S3 object storage for backups and application data
- **Docker Host**: N100 mini PC running various containerized services (always-on)
- **Primary K3s Node**: `mainkube` - VM on main PC with GPU passthrough for testing/development and AI
- **Network Infrastructure**: 
  - **UniFi Ultra**: Core router/firewall/controller
  - **UniFi Enterprise 16-Port PoE**: Managed switching with PoE+
  - **UniFi Access Points**: WiFi coverage
  - **5Gb Backbone**: WizDPI networking for high-speed inter-cluster communication
- **Storage**: Ceph cluster + TrueNAS ZFS + SMB/NFS shares

### Planned Hardware (K3s HA Expansion)
- **Worker Nodes**: Raspberry Pi cluster running Talos OS
- Control Plane Nodes: K3s VMs across all Proxmox cluster nodes for HA
  - Main PC VM: Primary control plane with GPU passthrough
  - XPS 15 VM: Secondary control plane node
  - Razer 15 VM: node
- **Load Balancing**: MetalLB for service distribution across K3s nodes
- **High Availability**: Multi-master K3s cluster architecture
- **Storage Integration**: Longhorn + TrueNAS S3 backend

## 🎮 NVIDIA GPU Support

### Prerequisites
1. NVIDIA drivers installed on host
2. NVIDIA Container Toolkit configured
3. Compatible GPU with driver version available at https://download.nvidia.com/XFree86/Linux-x86_64/

### Configuration

```bash
# Configure K3s with NVIDIA runtime
sudo nvidia-ctk runtime configure \
  --runtime=containerd \
  --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml

sudo systemctl restart k3s
```

### GPU-Enabled Applications
- ~~**Jellyfin**: Hardware transcoding for 4K media~~
- **Ollama**: Accelerated LLM inference  
- ~~**Steam Headless**: GPU-accelerated game streaming~~

## 🔒 Security & Access

### Secret Management
All sensitive data is encrypted using **SOPS** with **age** encryption:
~~- SMB/CIFS credentials for media storage~~
- Cloudflare tunnel certificates  
- Application secrets and API keys

### External Access
- **Cloudflare Tunnels** provide secure external access to select services
- **Tailscale** provides secure VPN access to the entire homelab network for personal use
- **Traefik** handles internal routing and SSL termination
- **Network isolation** via Kubernetes namespaces

## 💾 Storage Strategy

### Overview

The homelab uses a **TrueNAS-centric storage architecture** that simplifies operations by leveraging TrueNAS as the primary storage backend for all persistent data. Since application availability is tied to TrueNAS availability anyway (apps can't function without data access), this approach reduces complexity while maintaining reliability.

**📚 Documentation:**
- 🚀 **[Quick Start Guide](./docs/STORAGE-QUICK-START.md)** - Start here! 30-second decision tree and migration cheat sheet
- 📖 **[Architecture Strategy](./docs/STORAGE-ARCHITECTURE.md)** - Complete technical architecture and rationale
- 📋 **[Migration Guide](./docs/STORAGE-MIGRATION-GUIDE.md)** - Step-by-step migration instructions
- 🔧 **[Ceph Deployment](./docs/CEPH-DEPLOYMENT-GUIDE.md)** - Optional Ceph setup for VM HA (if needed)

### Storage Tiers

#### Tier 1: TrueNAS Primary Storage (Recommended)
- **Technology**: NFS/SMB from TrueNAS + S3 (MinIO)
- **Use Cases**: All persistent application data, media files, databases
- **Status**: ✅ In use for media (audiobookshelf)
- **Next Steps**: Migrate remaining apps from local-path to TrueNAS NFS

#### Tier 2: Node-Local Fast Storage
- **Technology**: K3s local-path provisioner
- **Use Cases**: Caches, temporary files, ephemeral data
- **Status**: ✅ In use for most apps
- **Recommendation**: Keep for caches, migrate persistent data to TrueNAS

#### Tier 3: Ceph Distributed Storage (Optional)
- **Technology**: Ceph RBD across 3 Proxmox nodes
- **Use Cases**: VM live migration, K3s control plane HA
- **Status**: 📋 Planned (optional)
- **Recommendation**: Deploy only if you need VM live migration

### Storage Classes

| Storage Class | Type | Primary Use | Current Status |
|---------------|------|-------------|----------------|
| `truenas-nfs` | TrueNAS NFS | **Recommended for all persistent data** | 📋 To be deployed |
| `local-path` | Node-local | Caches and ephemeral storage | ✅ In use |
| `longhorn` | Distributed | Legacy, being phased out | ⚠️ Not recommended |
| `ceph-rbd` | Ceph | VM storage (Proxmox only) | 🔮 Optional |
| `truenas-s3` | MinIO S3 | Backups and archives | 📋 Planned |

### Migration Plan

**Phase 1: Expand TrueNAS Integration** (Current)
- Deploy NFS CSI provisioner for dynamic PV provisioning
- Migrate apps from local-path to TrueNAS NFS
- Configure automated ZFS snapshots
- See: [docs/STORAGE-MIGRATION-GUIDE.md](./docs/STORAGE-MIGRATION-GUIDE.md)

**Phase 2: Implement S3 Object Storage**
- Deploy MinIO on TrueNAS Scale
- Configure backup pipeline with Velero/Restic
- Automated database backups to S3

**Phase 3: Ceph Deployment** (Optional)
- Deploy Ceph on Proxmox for VM live migration
- Only for infrastructure VMs, not application data
- See: [docs/CEPH-DEPLOYMENT-GUIDE.md](./docs/CEPH-DEPLOYMENT-GUIDE.md)

### Backup Strategy

**TrueNAS-Centric Approach:**
- **ZFS Snapshots**: Automated hourly/daily/weekly snapshots on TrueNAS
- **ZFS Replication**: Off-site replication to external backup
- **Application Backups**: Database dumps to S3 (daily)
- **GitOps**: All manifests and configs in Git with SOPS encryption
- **Proxmox VM snapshots**: Control plane VM backups

**Disaster Recovery RTO/RPO:**
- Single app failure: 5 min RTO, 0 RPO (data on TrueNAS)
- K3s node failure: 2 min RTO, 0 RPO (automatic rescheduling)
- TrueNAS failure: 4 hours RTO, 24 hours RPO (restore from backup)
- Complete cluster loss: 2 hours RTO, 0 RPO (rebuild from Git + TrueNAS)

## 🐛 Troubleshooting

### GPU Issues

```bash
# Verify GPU is available in cluster
kubectl describe nodes | grep nvidia.com/gpu

# Check NVIDIA runtime configuration
sudo nvidia-ctk runtime configure --runtime=containerd --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml
```

## 🔮 Roadmap

### Storage Architecture (In Progress)
- [x] **Storage architecture planning** - Comprehensive TrueNAS-centric strategy defined
- [ ] **Deploy NFS CSI provisioner** - Dynamic PV provisioning from TrueNAS
- [ ] **Migrate apps to TrueNAS NFS** - Move persistent data from local-path to TrueNAS
- [ ] **MinIO S3 deployment** - Object storage for backups and archives
- [ ] **Optional Ceph deployment** - VM live migration capability (if needed)

### Cluster Expansion
- [ ] **Add Raspberry Pi workers** - Deploy Talos OS on RPi cluster for HA
- [ ] **Add laptop control plane nodes** - Configure XPS 15 and Razer 15 as K3s masters/workers for HA control plane
- [ ] **MetalLB implementation** - Load balancer for service distribution across K3s nodes

### Infrastructure Improvements
- [ ] **Deploy Vault** - Centralized secret management across cluster
- [ ] **Docker container migration** - Move services from N100 mini PC to K3s cluster
- [ ] **Tailscale operator** - Native Kubernetes integration for VPN access
- [ ] **Production environment** - Create production overlay configurations
- [ ] **Storage monitoring dashboards** - Grafana dashboards for TrueNAS and Ceph metrics


---
