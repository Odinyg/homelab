# Storage Architecture Strategy

## Executive Summary

This document outlines the comprehensive storage strategy for the homelab Kubernetes cluster, optimizing the use of TrueNAS NAS and the 3-node Proxmox cluster with optional Ceph deployment.

**Key Principle**: Since TrueNAS holds all critical data, application availability is inherently tied to TrueNAS availability. This simplifies the architecture—we can leverage TrueNAS as the primary storage backend without complex high-availability requirements at the K3s level.

## Current Infrastructure

### Hardware
- **TrueNAS Scale NAS** (10.10.10.20)
  - N100-based system with ZFS
  - 4x 5Gb networking
  - Primary data repository
  - Already serving NFS shares
  
- **Proxmox Cluster** (3 nodes)
  - Main Station PC (RTX 3090)
  - XPS 15 (5Gb networking)
  - Razer 15 (5Gb networking)
  - Potential for Ceph distributed storage

- **K3s Cluster**
  - Currently single node ("mainkube")
  - Planned expansion to HA control plane
  - GPU passthrough for AI/ML workloads

### Current Storage Classes

| Storage Class | Type | Use Case | Current Status |
|---------------|------|----------|----------------|
| `local-path` | Node-local hostPath | Fast ephemeral storage, testing | ✅ In use |
| `longhorn` | Distributed block storage | Replicated persistent data | ⚠️ Limited use |
| `nfs` | TrueNAS NFS shares | Media files, large datasets | ✅ In use |
| `smb` | TrueNAS SMB/CIFS | Windows compatibility | 📋 Planned |
| `s3` | Object storage | Backups, archives | 📋 Planned |

## Recommended Storage Architecture

### Three-Tier Storage Strategy

#### Tier 1: TrueNAS Primary Storage (Recommended)
**Purpose**: All persistent application data and media

**Technology**: NFS/SMB from TrueNAS + potential S3 (MinIO on TrueNAS)

**Rationale**:
- Single source of truth for all data
- ZFS provides data integrity and snapshots
- Already proven reliable in production
- Application downtime during TrueNAS maintenance is acceptable since data access would be unavailable anyway
- Simplified backup strategy (backup TrueNAS, not each app)
- 5Gb networking provides excellent throughput

**Implementation**:
- Use NFS StorageClass for Kubernetes PVs
- Direct mount TrueNAS shares for media libraries
- Deploy MinIO on TrueNAS for S3-compatible object storage
- Configure per-application datasets on TrueNAS for granular snapshots

**Recommended Apps**:
- ✅ Media servers (Jellyfin, Audiobookshelf) - already using NFS
- ✅ Databases (PostgreSQL, Redis) - add to TrueNAS NFS
- ✅ Application data (Linkding, Karakeep) - migrate to NFS
- ✅ AI/ML models (Ollama) - large model storage on NFS
- ✅ Backup repositories - S3 backend on TrueNAS

#### Tier 2: Node-Local Fast Storage (Current: local-path)
**Purpose**: Temporary data, caches, and performance-critical workloads

**Technology**: K3s local-path provisioner (hostPath)

**Rationale**:
- Fastest possible I/O for application caches
- No network overhead
- Acceptable data loss on node failure (ephemeral by design)
- Good for development/testing

**Implementation**:
- Keep existing local-path storage class
- Use for application caches and temporary files
- Store session data and application state that can be rebuilt

**Recommended Apps**:
- ✅ Application caches (Redis cache layers)
- ✅ Temporary processing directories
- ✅ Build artifacts and CI/CD workspaces
- ✅ Testing/development environments

#### Tier 3: Ceph Distributed Storage (Optional)
**Purpose**: HA control plane and critical cluster services

**Technology**: Ceph RBD across 3 Proxmox nodes

**Rationale**:
- Provides HA for K3s control plane data
- Allows VM live migration across Proxmox nodes
- No single point of failure for cluster infrastructure
- **NOT recommended for application data** (TrueNAS is simpler and more reliable)

**Implementation** (if deployed):
- Deploy Ceph across all 3 Proxmox nodes
- Use Ceph for Proxmox VM storage only
- Use Rook-Ceph operator for K3s etcd backup storage (optional)
- **Do not** use for application persistent volumes

**Recommended Use Cases**:
- ✅ Proxmox VM disks for K3s control plane nodes
- ✅ HA control plane etcd data (via Rook-Ceph)
- ❌ Application persistent volumes (use TrueNAS instead)
- ❌ Media files (use TrueNAS NFS instead)

## Storage Decision Matrix

| Application Type | Recommended Storage | Rationale |
|------------------|---------------------|-----------|
| Media Files (video, audio, books) | TrueNAS NFS | Large files, shared access |
| Databases (PostgreSQL, MySQL) | TrueNAS NFS | Persistent, needs backups |
| **SQLite Databases** | **local-path or Longhorn** | **NFS locking issues, see note below** |
| Application Config | TrueNAS NFS | Persistent, small, needs backups |
| AI/ML Models | TrueNAS NFS | Large models, shared across pods |
| Application Caches | local-path | Fast, ephemeral, can rebuild |
| Session Storage | local-path | Fast, ephemeral, can rebuild |
| Backup Repository | TrueNAS S3 (MinIO) | Long-term storage, versioning |
| K3s Control Plane | Ceph (optional) | HA requirement, small size |

### ⚠️ Important: SQLite on NFS Considerations

**SQLite databases have known issues with NFS** due to file locking problems, especially during network interruptions or connection losses. This affects several common homelab applications:

**Apps using SQLite:**
- Linkding (bookmark manager)
- Audiobookshelf (metadata database)
- Many lightweight web apps

**The Problem:**
- NFS uses advisory locking, which SQLite relies on
- Network interruptions can cause lock timeouts
- Database corruption risk if locks fail during writes
- WAL mode doesn't fully solve the issue on NFS

**Recommended Solutions:**

#### Option 1: Use Local Storage for SQLite Apps (Recommended)
```yaml
# Keep SQLite databases on local-path or Longhorn
storageClassName: local-path  # or longhorn for replication
```

**Pros:**
- No NFS locking issues
- Better performance for SQLite
- Proven reliability

**Cons:**
- Tied to single node (local-path)
- Need manual backups
- Or use Longhorn for replication (adds complexity)

#### Option 2: Migrate to PostgreSQL
For critical apps like Linkding, consider migrating to PostgreSQL:
```yaml
# Run PostgreSQL on TrueNAS NFS (works great!)
# Configure app to use PostgreSQL instead of SQLite
```

**Pros:**
- PostgreSQL works perfectly on NFS
- Better for multi-node setups
- More robust for production

**Cons:**
- Requires database migration
- Additional operational complexity

#### Option 3: Use ReadWriteOnce + Node Affinity
Pin SQLite apps to specific nodes with local storage:
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - mainkube  # Pin to specific node
```

**Pros:**
- Works with single-replica apps
- Simpler than Longhorn

**Cons:**
- App unavailable if node goes down
- Manual migration if node changes

### Updated Recommendations for Known Apps

| App | Database | Recommended Storage | Alternative |
|-----|----------|---------------------|-------------|
| Linkding | SQLite | local-path + backups | Migrate to PostgreSQL on NFS |
| Audiobookshelf | SQLite (metadata) | local-path for config/metadata, NFS for media | Keep current setup |
| Ollama | File-based | TrueNAS NFS (models are just files) | N/A |
| Karakeep | Unknown | Verify database type first | If SQLite, use local-path |

## Migration Plan

### Phase 1: Expand TrueNAS Integration (Immediate)
1. **Create TrueNAS Datasets**
   ```
   /mnt/big/k8s-apps/
   ├── ollama/          # AI models and config
   ├── linkding/        # Bookmarks database
   ├── karakeep/        # Karaoke data
   ├── databases/       # PostgreSQL, Redis data
   └── backups/         # Application backups
   ```

2. **Configure NFS Exports**
   - One export per application or logical grouping
   - Set appropriate permissions (squash, sync options)
   - Enable NFS v4 for better performance

3. **Create Kubernetes StorageClasses**
   - Dedicated StorageClass for TrueNAS NFS
   - Dynamic provisioning using nfs-subdir-external-provisioner
   - Automatic PV creation for new PVCs

4. **Migrate Existing Applications**
   - Ollama: Move from local-path to TrueNAS NFS
   - Linkding: Move from local-path to TrueNAS NFS  
   - Karakeep: Move from local-path to TrueNAS NFS
   - Keep config/metadata on NFS, cache on local-path

### Phase 2: Implement S3 Object Storage (Near-term)
1. **Deploy MinIO on TrueNAS**
   - Run MinIO container on TrueNAS Scale
   - Backend: ZFS dataset for S3 data
   - Use for application backups and archives

2. **Configure S3 StorageClass**
   - Deploy CSI S3 driver in K8s
   - Configure backup tools (Velero, Restic)

3. **Setup Backup Pipeline**
   - Automated PostgreSQL dumps to S3
   - Application data snapshots to S3
   - ZFS snapshot replication

### Phase 3: Ceph Deployment (Optional, Long-term)
**Only deploy if you need:**
- VM live migration across Proxmox nodes
- HA for K3s control plane VMs
- Redundancy for critical cluster infrastructure

**Steps**:
1. Configure Ceph cluster across 3 Proxmox nodes
2. Create Ceph RBD pool for Proxmox VM storage
3. Migrate K3s control plane VMs to Ceph storage
4. Optionally deploy Rook-Ceph for K8s-native Ceph access

**Note**: For application data, continue using TrueNAS NFS. Ceph adds complexity without benefit for your use case.

## Implementation Examples

### TrueNAS NFS StorageClass (Recommended)

```yaml
# infrastructure/controllers/base/nfs-provisioner/storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: truenas-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.10.10.20
  share: /mnt/big/k8s-apps
  # Mount options for better performance
  mountOptions:
    - nfsvers=4.1
    - hard
    - timeo=600
    - retrans=2
    - rsize=1048576
    - wsize=1048576
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

### Application Migration Example: Ollama

**Before** (local-path):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ollama-pv
spec:
  capacity:
    storage: 1Ti
  storageClassName: local-path
  hostPath:
    path: /mnt/slow/ollama
```

**After** (TrueNAS NFS):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-models-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteMany  # Can be accessed from multiple nodes
  storageClassName: truenas-nfs
  resources:
    requests:
      storage: 500Gi
```

### Hybrid Storage Example: AI Workload

```yaml
# ollama-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
spec:
  template:
    spec:
      volumes:
        # Models on TrueNAS (persistent, large)
        - name: models
          persistentVolumeClaim:
            claimName: ollama-models-pvc  # truenas-nfs
        
        # Cache on local storage (fast, ephemeral)
        - name: cache
          persistentVolumeClaim:
            claimName: ollama-cache-pvc   # local-path
      
      containers:
        - name: ollama
          volumeMounts:
            - name: models
              mountPath: /root/.ollama/models
            - name: cache
              mountPath: /tmp/ollama-cache
```

## Backup and Disaster Recovery

### TrueNAS-Centric Backup Strategy

Since TrueNAS is the single source of truth:

1. **ZFS Snapshots** (Automated)
   - Hourly snapshots retained for 24 hours
   - Daily snapshots retained for 7 days
   - Weekly snapshots retained for 4 weeks
   - Monthly snapshots retained for 12 months

2. **ZFS Replication** (Recommended)
   - Replicate to off-site TrueNAS or external backup
   - Replicate critical datasets to cloud S3

3. **Application-Level Backups**
   - Database dumps to S3 (daily)
   - Application state exports to S3
   - Velero for K8s resource backups

4. **GitOps as Code**
   - All manifests in Git (already done)
   - SOPS-encrypted secrets in Git
   - Can rebuild entire cluster from Git + TrueNAS data

### Disaster Recovery Scenarios

| Scenario | Recovery Steps | RTO | RPO |
|----------|----------------|-----|-----|
| Single app failure | Redeploy from Git, data intact on TrueNAS | 5 min | 0 |
| K3s node failure | K8s reschedules pods, data intact on TrueNAS | 2 min | 0 |
| TrueNAS failure | Restore TrueNAS from backup/replication | 4 hours | 24 hours |
| Complete cluster loss | Rebuild K3s, apply manifests, mount TrueNAS | 2 hours | 0 (data) |

## Performance Considerations

### Network Performance
- 5Gb networking between TrueNAS and K3s nodes provides ~600 MB/s throughput
- NFS v4.1 with jumbo frames recommended
- Direct attach for media streaming (no switch bottleneck)

### Storage Performance Tiers
1. **Fastest**: local-path on NVMe (for caches)
2. **Fast**: TrueNAS NFS over 5Gb network
3. **Distributed**: Ceph (if deployed) - adds latency but provides redundancy

### Optimization Tips
- Use local-path for application caches and temporary data
- Use ReadWriteMany for shared access (multiple pods)
- Use ReadWriteOnce for exclusive access (better performance)
- Mount media libraries read-only when possible

## Security Considerations

### NFS Security
- Use NFS v4 with Kerberos (optional, complex)
- Restrict NFS exports by IP subnet (K3s nodes only)
- Use `root_squash` to prevent root access from clients
- Enable `sync` mode for data integrity

### S3 Security
- Generate unique access keys per application
- Use TLS for all S3 traffic
- Implement bucket policies for least privilege
- Enable versioning for backup buckets

### Ceph Security (if deployed)
- Enable Ceph authentication (CephX)
- Use separate pools for different security zones
- Encrypt data at rest (LUKS)

## Monitoring and Maintenance

### Storage Monitoring
- TrueNAS built-in SMART monitoring
- Prometheus metrics for NFS performance
- Grafana dashboards for storage utilization
- Alert on:
  - ZFS pool degradation
  - High storage usage (>80%)
  - NFS mount failures
  - Snapshot failures

### Maintenance Windows
- TrueNAS updates: Schedule during low-usage periods
- Expect applications to be unavailable during TrueNAS maintenance
- Use pod disruption budgets for graceful shutdowns
- Test NFS failover and recovery procedures

## Recommendations Summary

### Do This Now
1. ✅ **Expand TrueNAS NFS usage** - migrate apps from local-path to TrueNAS NFS
2. ✅ **Configure automated ZFS snapshots** on TrueNAS
3. ✅ **Deploy NFS CSI provisioner** for dynamic PV provisioning
4. ✅ **Document NFS mount points** for each application
5. ✅ **Setup off-site replication** for critical TrueNAS datasets

### Do This Soon
1. 📋 **Deploy MinIO on TrueNAS** for S3-compatible object storage
2. 📋 **Implement backup pipeline** with Velero or Restic
3. 📋 **Create storage monitoring dashboards** in Grafana
4. 📋 **Test disaster recovery procedures**

### Do This Later (Optional)
1. 🔮 **Deploy Ceph on Proxmox** - only if you need VM live migration or HA control plane
2. 🔮 **Implement Rook-Ceph** - only if you deploy Ceph and need K8s-native access
3. 🔮 **Multi-site replication** - if you add a second location

### Don't Do This
1. ❌ **Don't use Longhorn for application data** - TrueNAS NFS is simpler and more reliable
2. ❌ **Don't store critical data on local-path** - it's ephemeral and tied to a single node
3. ❌ **Don't replicate between storage backends** - pick one (TrueNAS) and stick with it
4. ❌ **Don't deploy Ceph for application data** - unnecessary complexity vs TrueNAS

## Conclusion

The recommended architecture leverages your existing TrueNAS as the primary storage backend, with optional Ceph deployment for Proxmox VM high availability. This approach:

- **Simplifies** the storage stack (one primary backend)
- **Reduces** complexity and failure modes
- **Aligns** with your stated requirements (apps depend on TrueNAS anyway)
- **Leverages** existing reliable infrastructure (TrueNAS + ZFS)
- **Provides** clear migration path from current state
- **Enables** future expansion (S3, Ceph for VMs)

The key insight: Since your data is centralized on TrueNAS and applications cannot function without access to that data, it makes sense to embrace TrueNAS as the primary storage tier rather than adding complexity with multiple storage backends.

## Next Steps

1. Review this architecture plan
2. Create TrueNAS datasets for K8s applications
3. Deploy NFS CSI provisioner to K8s cluster
4. Begin migrating applications from local-path to TrueNAS NFS
5. Document per-application storage requirements
6. Test backup and recovery procedures

See `docs/STORAGE-MIGRATION-GUIDE.md` for detailed migration instructions.
