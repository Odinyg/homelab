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

## SQLite Disaster Recovery Strategies

### The Node Failure Problem

When SQLite databases are on local-path storage:
- ✅ **Safe from NFS corruption** - No locking issues
- ❌ **Not resilient to node failure** - If node dies, app can't restart on another node
- ❌ **Data loss risk** - If node disk fails completely, database is lost

**Question:** How to protect SQLite databases from node failures without using NFS?

### Solution Comparison

| Strategy | Data Safety | Node Failure Recovery | Complexity | Cost |
|----------|-------------|----------------------|------------|------|
| **local-path + backups** | ⚠️ Manual restore | ❌ Manual | Low | Low |
| **Longhorn replication** | ✅ Automatic | ✅ Automatic | Medium | Medium |
| **PostgreSQL migration** | ✅ NFS + backups | ✅ Automatic | High | Low |
| **Regular snapshots** | ✅ Point-in-time | ⚠️ Data loss window | Low | Low |

### Recommended Approach: Longhorn for Critical SQLite Apps

**When to use Longhorn:**
- SQLite app is critical (downtime/data loss unacceptable)
- Need automatic failover to other nodes
- Willing to accept moderate complexity

**How Longhorn solves the problem:**
1. **Replicates SQLite database** across 3 nodes (configurable)
2. **No NFS involved** - Block storage, not file-based
3. **Automatic failover** - If node fails, pod reschedules with data intact
4. **No corruption risk** - Direct block replication, not file locking

#### Longhorn Configuration for SQLite

```yaml
# infrastructure/controllers/base/longhorn/storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-sqlite
parameters:
  numberOfReplicas: "3"  # Replicate across 3 nodes
  staleReplicaTimeout: "2880"  # 48 hours
  fromBackup: ""
  fsType: "ext4"
  dataLocality: "disabled"  # Allow scheduling on any node
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

#### Using Longhorn for SQLite Apps

**Linkding example:**
```yaml
# apps/base/linkding/storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data-pvc
  namespace: linkding
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-sqlite  # ✅ Replicated across nodes
  resources:
    requests:
      storage: 5Gi
```

**Deployment configuration:**
```yaml
# apps/base/linkding/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1  # Single replica (SQLite limitation)
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      # No node affinity needed - Longhorn handles failover
      containers:
      - name: linkding
        image: sissbruecker/linkding:latest
        volumeMounts:
        - name: data
          mountPath: /etc/linkding/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: linkding-data-pvc
```

**What happens when node fails:**
1. Kubernetes detects node failure
2. Pod is rescheduled to healthy node
3. Longhorn attaches replicated volume to new node
4. App starts with data intact (0 data loss)
5. Downtime: ~2-5 minutes for pod reschedule

### Alternative: Automated Backup Strategy

**For less critical SQLite apps** where some downtime is acceptable:

#### Option 1: Backup CronJob to TrueNAS

```yaml
# apps/base/linkding/backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: linkding-backup
  namespace: linkding
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: busybox:latest
            command:
            - /bin/sh
            - -c
            - |
              # Create timestamped backup
              BACKUP_FILE="/backup/linkding-$(date +%Y%m%d-%H%M%S).tar.gz"
              tar czf "$BACKUP_FILE" -C /data .
              
              # Keep only last 14 backups (7 days at 2/day)
              cd /backup
              ls -t linkding-*.tar.gz | tail -n +15 | xargs -r rm
              
              echo "Backup completed: $BACKUP_FILE"
              ls -lh /backup/
            volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
            - name: backup
              mountPath: /backup
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: linkding-data-pvc
          - name: backup
            nfs:
              server: 10.10.10.20
              path: /mnt/big/k8s-apps/backups/linkding
```

**Recovery process if node fails:**
```bash
# 1. Deploy to new node (automatic)
kubectl apply -k apps/staging/linkding/

# 2. Restore latest backup
LATEST_BACKUP=$(ssh truenas "ls -t /mnt/big/k8s-apps/backups/linkding/*.tar.gz | head -1")
kubectl exec -n linkding deployment/linkding -- sh -c "
  cd /etc/linkding/data
  rm -rf *
  wget -qO- http://truenas.local/backups/linkding/$(basename $LATEST_BACKUP) | tar xzf -
"

# 3. Restart pod
kubectl rollout restart deployment/linkding -n linkding
```

**Downtime:** Depends on RTO (15 minutes with automation)  
**Data loss:** Up to 6 hours (backup interval)

#### Option 2: Velero for Cluster Backup

```yaml
# Install Velero with TrueNAS S3 backend
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket k8s-backups \
  --secret-file ./minio-credentials \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://truenas.local:9000

# Schedule Linkding backup
velero schedule create linkding-backup \
  --schedule="0 */4 * * *" \
  --include-namespaces linkding \
  --default-volumes-to-fs-backup
```

**Recovery:**
```bash
velero restore create --from-backup linkding-backup-20241114
```

### Decision Matrix: Which Strategy?

| App Criticality | Data Loss Tolerance | Recommended Strategy | Estimated RTO | RPO |
|-----------------|---------------------|---------------------|---------------|-----|
| **Critical** (user data) | None | Longhorn replication | 2-5 min | 0 |
| **Important** (can rebuild) | < 6 hours | Backup CronJob (6h) | 15 min | 6 hours |
| **Low** (testing/dev) | 24 hours | Daily backup | 30 min | 24 hours |
| **Best** (production) | None | Migrate to PostgreSQL | 2 min | 0 |

### Recommended Per App

| App | Criticality | Strategy | Why |
|-----|-------------|----------|-----|
| **Linkding** | High (bookmarks) | **Longhorn replication** | User data, frequent writes |
| **Audiobookshelf** | Medium (metadata) | Backup CronJob (4h) | Metadata can be rebuilt from media |
| **Dev/Test apps** | Low | Daily backup or local-path only | Acceptable to rebuild |

### Implementation Steps

**For Longhorn approach:**
```bash
# 1. Ensure Longhorn is deployed
kubectl get storageclass longhorn

# 2. Create SQLite-specific StorageClass
kubectl apply -f infrastructure/controllers/base/longhorn/storage-class-sqlite.yaml

# 3. Update app to use Longhorn
# Edit apps/base/linkding/storage.yaml
# Change storageClassName to: longhorn-sqlite

# 4. Migrate data
kubectl scale deployment linkding -n linkding --replicas=0
# Copy data from local-path PV to new Longhorn PV
kubectl apply -k apps/staging/linkding/
kubectl scale deployment linkding -n linkding --replicas=1
```

**For backup approach:**
```bash
# 1. Create backup CronJob
kubectl apply -f apps/base/linkding/backup-cronjob.yaml

# 2. Verify backups working
kubectl get cronjobs -n linkding
kubectl logs -n linkding job/linkding-backup-<id>

# 3. Test restore procedure
# (Document in runbook)
```

### Monitoring and Alerts

**For Longhorn:**
```yaml
# Alert on replica degradation
- alert: LonghornVolumeReplicaDegraded
  expr: longhorn_volume_replica_count < 3
  for: 5m
  annotations:
    summary: "Longhorn volume {{ $labels.volume }} has degraded replicas"
```

**For backups:**
```yaml
# Alert on backup failure
- alert: BackupJobFailed
  expr: kube_job_status_failed{namespace="linkding",job_name=~"linkding-backup.*"} > 0
  annotations:
    summary: "Linkding backup job failed"
```

### Cost-Benefit Analysis

**Longhorn Replication:**
- **Cost**: 3x storage usage (replication factor 3), CPU/network for replication
- **Benefit**: Automatic failover, 0 data loss, 2-5 min RTO
- **Best for**: Critical SQLite apps with frequent writes

**Backup Strategy:**
- **Cost**: Minimal (storage for backups only), negligible CPU
- **Benefit**: Simple, reliable, good enough for most cases
- **Best for**: Less critical apps, read-heavy workloads

**PostgreSQL Migration:**
- **Cost**: One-time migration effort, slightly more RAM
- **Benefit**: Best long-term solution, NFS compatible, scales better
- **Best for**: Apps with PostgreSQL support, future-proof

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
