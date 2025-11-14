# Storage Quick Start Guide

**TL;DR**: Use TrueNAS for everything persistent. Skip Ceph unless you need VM live migration.

## 30-Second Decision Tree

```
Do you need to store data persistently?
├─ YES → Use TrueNAS NFS
│   ├─ Databases: ✅ TrueNAS NFS
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
| Media files (video, audio) | TrueNAS NFS | Ceph, Longhorn | Large files, already there |
| Application configs | TrueNAS NFS | local-path | Persistent, small, important |
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

1. **Don't use multiple storage backends for same data**
   - Bad: App config on local-path, database on Longhorn, media on NFS
   - Good: Everything persistent on TrueNAS NFS, only caches on local-path

2. **Don't deploy Ceph for application data**
   - Bad: Migrate apps from local-path to Ceph
   - Good: Use Ceph only for Proxmox VMs (if needed)

3. **Don't store persistent data on local-path**
   - Bad: Database on local-path (lost on node failure)
   - Good: Database on TrueNAS NFS (survives node failure)

4. **Don't skip backups**
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

**A: Phase it out.**

Longhorn adds complexity without benefits over TrueNAS:
- Uses node local disks (limited capacity)
- Replicates over network (slower than TrueNAS)
- Another system to manage and debug
- TrueNAS provides same features + ZFS benefits

Recommendation: Migrate from Longhorn to TrueNAS NFS.

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
- ✅ TrueNAS NFS for all persistent data
- ✅ local-path for caches and ephemeral data
- ⚠️ Ceph only if you need VM live migration (optional)
- ❌ Phase out Longhorn
- ❌ Don't use multiple backends for same data

**Why It Works:**
- Simpler architecture
- Single source of truth
- Aligns with reality (apps depend on TrueNAS)
- Proven reliable (ZFS + NFS)
- Easy to backup and restore

**Bottom Line:**
Don't overthink it. Use TrueNAS for everything important. You already have it, it's reliable, and it's fast enough.
