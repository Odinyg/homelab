# Storage Quick Start Guide

**TL;DR**: Use TrueNAS for everything persistent. Skip Ceph unless you need VM live migration.

## 30-Second Decision Tree

```
Does your app use SQLite database?
├─ YES → ⚠️ Use local-path (NFS breaks SQLite!)
│   ├─ Linkding: ✅ local-path
│   ├─ Audiobookshelf config: ✅ local-path
│   └─ Other SQLite apps: ✅ local-path
│
└─ NO → Is data persistent?
    ├─ YES → Use TrueNAS NFS
    │   ├─ PostgreSQL/MySQL: ✅ TrueNAS NFS
    │   ├─ Media files: ✅ TrueNAS NFS
    │   ├─ App configs: ✅ TrueNAS NFS
    │   └─ AI/ML models: ✅ TrueNAS NFS
    │
    └─ NO → Use local-path
        ├─ Caches: ✅ local-path
        ├─ Temp files: ✅ local-path
        └─ Session data: ✅ local-path

Do you need to live migrate VMs between Proxmox nodes?
├─ YES → Deploy Ceph (Proxmox only, not for apps)
└─ NO → Skip Ceph (you probably don't need it)
```

## Storage Decision Matrix

| What are you storing? | Use this | Not this | Why? |
|------------------------|----------|----------|------|
| Database (PostgreSQL, MySQL) | TrueNAS NFS | local-path, Ceph | Persistent, needs backups |
| **SQLite databases** ⚠️ | **local-path** | **TrueNAS NFS** | **NFS locking issues** |
| Media files (video, audio) | TrueNAS NFS | Ceph, Longhorn | Large files, already there |
| Application configs (non-SQLite) | TrueNAS NFS | local-path | Persistent, small, important |
| AI/ML models | TrueNAS NFS | local-path | Large, shared, persistent |
| Redis cache | local-path | TrueNAS NFS | Ephemeral, speed matters |
| Session storage | local-path | TrueNAS NFS | Ephemeral, can rebuild |
| Backups | TrueNAS S3 | Longhorn | Long-term, versioning |
| K3s control plane VMs | Ceph (optional) | N/A | Only if you want live migration |

## Why TrueNAS for Everything?

### The Core Insight

**Your apps can't work without TrueNAS anyway.**

If TrueNAS goes down, your apps lose access to data regardless of where they run. So it makes sense to store everything there:

- ✅ **Simpler** - One storage system to manage
- ✅ **Reliable** - ZFS is proven and battle-tested
- ✅ **Fast** - 5Gb networking is plenty
- ✅ **Backups** - ZFS snapshots are built-in
- ✅ **Recovery** - Restore TrueNAS = restore everything

### What About High Availability?

**You don't need complex HA for a homelab.**

Think about it:
- TrueNAS failure is rare (it's stable)
- When it fails, apps can't access data anyway
- Downtime of a few hours for TrueNAS restore is acceptable
- GitOps + TrueNAS = fast cluster rebuild

**vs. Complex HA setup:**
- Multiple storage backends (TrueNAS + Ceph + Longhorn)
- More components = more failure modes
- More operational overhead
- More things to monitor and debug

## ⚠️ Critical Exception: SQLite Databases

**SQLite does NOT work reliably on NFS!**

SQLite databases can experience corruption due to NFS file locking issues, especially during network interruptions or connection losses during writes.

### Apps That Use SQLite
- **Linkding** - Bookmark manager
- **Audiobookshelf** - Metadata database (config volume)
- Many other lightweight web apps

### What To Do

**For SQLite apps, use one of these approaches:**

1. **Keep on local-path** (Simplest)
   ```yaml
   storageClassName: local-path
   ```
   - ✅ Reliable, no NFS issues
   - ❌ Tied to single node
   - ✅ Use regular backups to TrueNAS

2. **Use Longhorn** (For replication)
   ```yaml
   storageClassName: longhorn
   ```
   - ✅ Replicated across nodes
   - ❌ More complex than local-path
   - ✅ Better for high-value data

3. **Migrate to PostgreSQL** (Best long-term)
   - Run PostgreSQL on TrueNAS NFS (works great!)
   - Migrate app data from SQLite to PostgreSQL
   - ✅ Scales better, more reliable

### What if Node with SQLite Dies?

**Problem:** SQLite on local-path means data tied to one node. If node dies, app can't start elsewhere.

**Solution 1: Longhorn Replication** (Best for critical apps)
```yaml
storageClassName: longhorn  # Replicates across 3 nodes
```
- ✅ **Automatic failover** - Pod reschedules to healthy node with data intact
- ✅ **0 data loss** - Longhorn keeps 3 copies across nodes
- ✅ **No NFS** - Block storage, no file locking issues
- ⚠️ **Cost**: 3x storage usage, moderate complexity
- **RTO**: 2-5 minutes
- **RPO**: 0 (no data loss)

**Solution 2: Automated Backups** (Good for less critical apps)
```yaml
# CronJob backing up every 6 hours to TrueNAS
storageClassName: local-path
```
- ✅ **Simple** - Just a backup CronJob
- ✅ **Low cost** - Only backup storage needed
- ⚠️ **Manual restore** - Need to restore from backup
- ⚠️ **Data loss window** - Lose up to 6 hours of data
- **RTO**: 15 minutes (with automation)
- **RPO**: 6 hours (backup interval)

**Solution 3: Migrate to PostgreSQL** (Best long-term)
```yaml
# Run PostgreSQL on TrueNAS NFS
# Change app to use PostgreSQL instead of SQLite
```
- ✅ **Works on NFS** - No locking issues
- ✅ **Scales better** - Multi-node compatible
- ⚠️ **Migration effort** - One-time data migration
- **RTO**: 2 minutes
- **RPO**: 0 (no data loss)

### Recommended Per App

| App | Criticality | Strategy | Reason |
|-----|-------------|----------|--------|
| **Linkding** | High (bookmarks) | **Longhorn replication** | User data, frequent writes |
| **Audiobookshelf** | Medium | Backup every 4h | Metadata can rebuild from media |
| **Dev/Test** | Low | local-path only | OK to rebuild |

**Bottom line**: Critical SQLite apps → Longhorn. Less critical → local-path + backups. Best → PostgreSQL.

## Three-Step Getting Started

### Step 1: Create TrueNAS Datasets

```bash
# SSH to TrueNAS
zfs create tank/k8s-apps
zfs create tank/k8s-apps/ollama
zfs create tank/k8s-apps/linkding
zfs create tank/k8s-apps/databases
```

### Step 2: Export via NFS

**TrueNAS Web UI:**
- Sharing → Unix Shares (NFS) → Add
- Path: `/mnt/big/k8s-apps`
- Network: Your K3s subnet
- Enable NFSv4

### Step 3: Deploy NFS Provisioner

```bash
# Clone this repo, then:
kubectl apply -k infrastructure/controllers/staging/nfs-provisioner/

# Verify
kubectl get storageclass truenas-nfs
```

**Done!** Now your apps can use `truenas-nfs` StorageClass.

## Migration Cheat Sheet

### Migrate Any App to TrueNAS NFS

```bash
# 1. Backup
kubectl scale deployment <app> -n <namespace> --replicas=0
sudo tar -czf /tmp/<app>-backup.tar.gz /var/lib/rancher/k3s/storage/<app>

# 2. Copy to TrueNAS
sudo mount -t nfs 10.10.10.20:/mnt/big/k8s-apps /mnt/truenas
sudo rsync -avP /var/lib/rancher/k3s/storage/<app>/ /mnt/truenas/<app>/
sudo umount /mnt/truenas

# 3. Update PVC to use truenas-nfs
# Edit apps/base/<app>/storage.yaml
# Change storageClassName to: truenas-nfs

# 4. Apply and verify
kubectl delete pvc <old-pvc> -n <namespace>
kubectl apply -k apps/staging/<app>/
kubectl get pods -n <namespace> -w
```

## Common Pitfalls

### ❌ Don't Do This

1. **Don't put SQLite databases on NFS** ⚠️
   - Bad: Linkding with SQLite on TrueNAS NFS (corruption risk!)
   - Good: Keep SQLite apps on local-path or Longhorn

2. **Don't use multiple storage backends for same data**
   - Bad: App config on local-path, database on Longhorn, media on NFS
   - Good: Pick one backend per data type (SQLite → local-path, rest → NFS)

3. **Don't deploy Ceph for application data**
   - Bad: Migrate apps from local-path to Ceph
   - Good: Use Ceph only for Proxmox VMs (if needed)

4. **Don't store non-SQLite persistent data on local-path**
   - Bad: PostgreSQL database on local-path (lost on node failure)
   - Good: PostgreSQL on TrueNAS NFS (survives node failure)
   - Exception: SQLite must stay on local-path due to NFS locking issues

5. **Don't skip backups**
   - Bad: Trust single storage system
   - Good: ZFS snapshots + off-site replication

### ✅ Do This Instead

1. **Use TrueNAS for all persistent data**
   - Databases, configs, media, AI models → TrueNAS NFS
   - Caches, temp files, sessions → local-path

2. **Configure automated snapshots**
   - TrueNAS → Tasks → Periodic Snapshots
   - Hourly (keep 24), Daily (keep 7), Weekly (keep 4)

3. **Test your backups**
   - Restore a snapshot monthly
   - Document recovery procedures
   - Time how long a full rebuild takes

4. **Monitor storage health**
   - TrueNAS SMART monitoring
   - Prometheus metrics for NFS
   - Grafana dashboards for usage

## FAQ

### Q: Should I use Ceph?

**A: Probably not.**

Only deploy Ceph if you specifically need:
- VM live migration between Proxmox nodes
- Zero-downtime Proxmox node maintenance

For app data, TrueNAS is simpler and better.

### Q: What about Longhorn?

**A: Keep it for SQLite apps, phase out for others.**

**Use Longhorn for:**
- ✅ Apps with SQLite databases (if you need replication)
- ✅ Apps where you need cross-node failover

**Don't use Longhorn for:**
- ❌ PostgreSQL/MySQL databases (use TrueNAS NFS)
- ❌ Media files (use TrueNAS NFS)
- ❌ Apps that work fine on local-path

Longhorn's main value: Replicating SQLite databases safely (unlike NFS).

### Q: What's wrong with SQLite on NFS?

**A: File locking issues cause corruption.**

SQLite uses file-level locking that doesn't work reliably over NFS:
- Network interruptions can cause lock timeouts
- Lost connections during writes risk corruption
- WAL mode doesn't fully solve it on NFS

**Apps affected:** Linkding, Audiobookshelf (metadata), many others.

**Solutions:**
1. Keep SQLite apps on local-path + backup to NFS
2. Use Longhorn for replication (if needed)
3. Migrate app to PostgreSQL (best long-term)

### Q: What if my node with SQLite database dies?

**A: Three options, depending on criticality.**

**Critical apps (can't lose data):**
- ✅ Use **Longhorn replication** across 3 nodes
- Pod automatically reschedules to healthy node
- Data intact via replicated volume
- RTO: 2-5 minutes, RPO: 0 (no data loss)

**Important apps (some data loss OK):**
- ✅ Use **automated backups** to TrueNAS (CronJob every 4-6 hours)
- Restore from latest backup when node fails
- RTO: 15 minutes, RPO: 4-6 hours

**Low priority apps:**
- ✅ Accept rebuild from scratch or daily backup
- RTO: 30 minutes, RPO: 24 hours

**Example Longhorn setup for Linkding:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data-pvc
spec:
  storageClassName: longhorn  # Replicates across nodes
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**What happens when node fails:**
1. Kubernetes detects node down
2. Pod reschedules to healthy node (30-60 seconds)
3. Longhorn attaches volume with replicated data
4. App starts with all data intact
5. Total downtime: 2-5 minutes

### Q: Won't NFS be slow?

**A: No, it's fast enough.**

- 5Gb networking = ~600 MB/s throughput
- NFS v4.1 with large buffers is efficient
- Most apps are not storage-limited
- Your current audiobookshelf uses NFS just fine

For truly latency-sensitive workloads (rare), use local-path for cache layer + TrueNAS for persistent data.

### Q: What if TrueNAS fails?

**A: Restore from backup.**

- ZFS snapshots protect against data corruption
- Off-site replication protects against hardware failure
- GitOps protects against config loss
- Total rebuild time: ~2-4 hours

Acceptable downtime for a homelab. Much simpler than managing complex HA storage.

### Q: Can I use SMB instead of NFS?

**A: Yes, but NFS is better for Linux.**

- NFS has better Kubernetes integration
- NFS has better performance for Linux
- SMB is fine for Windows-only apps

Recommendation: Use NFS for K8s, SMB for Windows clients.

### Q: How do I back up TrueNAS?

**A: Multiple layers:**

1. **ZFS snapshots** (on TrueNAS itself) - protect against accidental deletion
2. **ZFS replication** (to another TrueNAS or server) - protect against hardware failure
3. **Cloud backup** (to Backblaze B2, AWS S3) - protect against site loss
4. **GitOps** (this repo) - rebuild cluster config

## Next Steps

Ready to implement? Follow this order:

1. **Read**: [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md) - Understand the strategy
2. **Plan**: Identify which apps to migrate first (start with low-risk)
3. **Prepare**: Create TrueNAS datasets and NFS exports
4. **Deploy**: Install NFS CSI provisioner
5. **Migrate**: Follow [STORAGE-MIGRATION-GUIDE.md](./STORAGE-MIGRATION-GUIDE.md)
6. **Monitor**: Set up Grafana dashboards
7. **Test**: Verify backups and recovery procedures

## Getting Help

**Documentation:**
- [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md) - Complete architecture
- [STORAGE-MIGRATION-GUIDE.md](./STORAGE-MIGRATION-GUIDE.md) - Migration steps
- [CEPH-DEPLOYMENT-GUIDE.md](./CEPH-DEPLOYMENT-GUIDE.md) - Optional Ceph setup

**Still confused?**
- Review your current app storage configurations
- Check which apps use local-path vs NFS currently
- Start with one low-risk app (like linkding)
- Test thoroughly before migrating critical apps

## Summary

**The Strategy:**
- ✅ TrueNAS NFS for all persistent data (PostgreSQL, media, configs, AI models)
- ⚠️ **Exception: SQLite apps on local-path or Longhorn** (NFS locking issues!)
- ✅ local-path for caches and ephemeral data
- ⚠️ Keep Longhorn for SQLite apps needing replication (optional)
- ⚠️ Ceph only if you need VM live migration (optional)
- ❌ Don't use multiple backends for same data type

**Critical Rules:**
1. **Never** put SQLite databases on NFS (corruption risk)
2. **Always** use TrueNAS NFS for PostgreSQL/MySQL
3. **Always** keep media files on TrueNAS NFS
4. **Always** backup SQLite apps from local-path to TrueNAS

**Why It Works:**
- Simpler architecture (mostly single backend)
- SQLite safety (avoid NFS locking issues)
- Aligns with reality (apps depend on TrueNAS)
- Proven reliable (ZFS + NFS for most things)
- Easy to backup and restore

**Bottom Line:**
Use TrueNAS NFS for everything EXCEPT SQLite databases. Keep SQLite apps on local-path with backups to TrueNAS.
