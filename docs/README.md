# Storage Documentation Index

This directory contains comprehensive documentation for the homelab storage architecture strategy.

## 🚀 Start Here

**New to the storage architecture?** Start with the Quick Start Guide:

👉 **[STORAGE-QUICK-START.md](./STORAGE-QUICK-START.md)** - 30-second decision tree and cheat sheet

## 📚 Complete Documentation

### 1. Quick Start Guide
**File**: [STORAGE-QUICK-START.md](./STORAGE-QUICK-START.md) (8KB)

**What's inside:**
- 30-second decision tree (use this first!)
- Storage decision matrix by app type
- 3-step getting started guide
- Migration cheat sheet
- Common pitfalls and FAQ
- Summary and next steps

**Who should read this:** Everyone! Read this first to understand the basics.

### 2. Architecture Strategy
**File**: [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md) (14KB)

**What's inside:**
- Complete architecture overview
- Three-tier storage strategy
- Storage decision matrix
- Migration plan (3 phases)
- Implementation examples with YAML
- Backup and disaster recovery
- Performance considerations
- Security guidelines
- Monitoring and maintenance

**Who should read this:** Anyone implementing the storage architecture or wanting deep technical details.

### 3. Migration Guide
**File**: [STORAGE-MIGRATION-GUIDE.md](./STORAGE-MIGRATION-GUIDE.md) (17KB)

**What's inside:**
- Step-by-step TrueNAS setup
- NFS CSI provisioner deployment
- Application migration procedures
- Example migrations (Ollama, Linkding, Karakeep)
- Troubleshooting guide
- Rollback procedures
- Validation checklist

**Who should read this:** Anyone actively migrating applications to TrueNAS NFS.

### 4. Ceph Deployment Guide
**File**: [CEPH-DEPLOYMENT-GUIDE.md](./CEPH-DEPLOYMENT-GUIDE.md) (14KB)

**What's inside:**
- Decision point: Do you need Ceph?
- Architecture options (Proxmox integrated, Rook, standalone)
- Proxmox integrated Ceph deployment
- Monitoring and maintenance
- Performance tuning
- Troubleshooting
- Cost-benefit analysis

**Who should read this:** Only if you're considering Ceph deployment for VM live migration.

## 🎯 Quick Reference

### Storage Decision Tree

```
┌─────────────────────────────────────────┐
│ What are you storing?                   │
└─────────────────────────────────────────┘
               │
               ├─ Persistent data? ─────────► TrueNAS NFS
               │  (databases, configs, media)
               │
               ├─ Ephemeral data? ─────────► local-path
               │  (caches, temp files)
               │
               └─ VM storage? ─────────────┬─► Local disk (default)
                                           └─► Ceph (if you need live migration)
```

### Storage Classes Summary

| Storage Class | Use For | Don't Use For |
|---------------|---------|---------------|
| `truenas-nfs` | Databases, media, app configs, AI models | Caches, temp files |
| `local-path` | Caches, temp files, session storage | Databases, important data |
| `ceph-rbd` | Proxmox VM disks (optional) | Application data |
| `longhorn` | Nothing (phase out) | Anything |

### When to Use What

- **99% of apps**: TrueNAS NFS for data + local-path for cache
- **Proxmox VMs**: Local disk (or Ceph if you need live migration)
- **Backups**: TrueNAS S3 (MinIO)

## 📋 Implementation Checklist

Ready to implement? Follow this order:

### Phase 1: TrueNAS Setup
- [ ] Read Quick Start Guide
- [ ] Create TrueNAS datasets (`/mnt/big/k8s-apps/...`)
- [ ] Configure NFS exports
- [ ] Test NFS access from K3s node
- [ ] Set up ZFS snapshots

### Phase 2: Deploy NFS Provisioner
- [ ] Create NFS provisioner manifests
- [ ] Deploy to K3s cluster
- [ ] Verify StorageClass created
- [ ] Test dynamic PV provisioning

### Phase 3: Migrate Applications
- [ ] Start with low-risk app (e.g., linkding)
- [ ] Follow migration guide for each app
- [ ] Test thoroughly before moving to next
- [ ] Update documentation

### Phase 4: Configure Backups
- [ ] Verify ZFS snapshots working
- [ ] Set up off-site replication
- [ ] Test restore procedures
- [ ] Document recovery steps

### Phase 5: Monitoring
- [ ] Set up Grafana dashboards
- [ ] Configure alerts
- [ ] Monitor storage usage
- [ ] Regular health checks

## 🔍 Finding What You Need

### I want to...

- **Understand the overall strategy** → Read [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md)
- **Get started quickly** → Read [STORAGE-QUICK-START.md](./STORAGE-QUICK-START.md)
- **Migrate an app** → Follow [STORAGE-MIGRATION-GUIDE.md](./STORAGE-MIGRATION-GUIDE.md)
- **Deploy Ceph** → Check [CEPH-DEPLOYMENT-GUIDE.md](./CEPH-DEPLOYMENT-GUIDE.md)
- **Make a quick decision** → Use decision tree in Quick Start Guide
- **Troubleshoot an issue** → Check troubleshooting sections in relevant guide

### I'm looking for...

- **NFS setup instructions** → Migration Guide, Step 1-2
- **PVC examples** → Architecture Strategy, Implementation Examples
- **Backup strategy** → Architecture Strategy, Backup and DR section
- **Performance tuning** → Architecture Strategy, Performance section
- **Security best practices** → Architecture Strategy, Security section
- **Ceph decision criteria** → Ceph Guide, Decision Point section

## 🤔 Common Questions

### "Should I use Ceph?"

**Short answer:** Probably not.

**Long answer:** Only if you specifically need VM live migration between Proxmox nodes. For application data, TrueNAS NFS is simpler and better. See [CEPH-DEPLOYMENT-GUIDE.md](./CEPH-DEPLOYMENT-GUIDE.md) for detailed cost-benefit analysis.

### "What about Longhorn?"

**Recommendation:** Phase it out. TrueNAS provides the same benefits (replication, backups) with less complexity. See [STORAGE-QUICK-START.md](./STORAGE-QUICK-START.md) FAQ section.

### "Is NFS fast enough?"

**Yes.** Your 5Gb networking provides ~600 MB/s throughput, which is plenty for most apps. Current audiobookshelf already uses NFS successfully. See [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md) Performance section.

### "What if TrueNAS fails?"

**Restore from backup.** With ZFS snapshots + off-site replication, recovery takes ~2-4 hours. This is acceptable downtime for a homelab and much simpler than managing complex HA storage. See [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md) Disaster Recovery section.

## 📞 Getting Help

**If you're stuck:**

1. Check the relevant documentation above
2. Review the troubleshooting sections
3. Verify your setup against the examples
4. Check the FAQ sections

**Documentation structure:**
- Quick Start = Fast answers and cheat sheets
- Architecture = Deep technical understanding
- Migration Guide = Step-by-step procedures
- Ceph Guide = Optional advanced topic

## 🎓 Learning Path

**Beginner:** Just getting started?
1. Read Quick Start Guide cover to cover
2. Skim Architecture Strategy (understand the "why")
3. Try migrating one low-risk app following Migration Guide

**Intermediate:** Ready to migrate all apps?
1. Deep dive into Architecture Strategy
2. Follow Migration Guide step-by-step
3. Set up monitoring and backups

**Advanced:** Want to optimize or add Ceph?
1. Study performance tuning in Architecture Strategy
2. Review Ceph cost-benefit analysis
3. Plan Ceph deployment if benefits outweigh costs

## 📊 Documentation Stats

- **Total documentation**: 53KB
- **Total pages**: 4 comprehensive guides
- **Code examples**: 20+ YAML examples
- **Decision matrices**: 3 tables
- **Troubleshooting scenarios**: 10+ covered
- **Reading time**: 
  - Quick Start: 15 minutes
  - Architecture: 45 minutes
  - Migration Guide: 60 minutes
  - Ceph Guide: 45 minutes

## ✅ Document Status

All documents are **complete and ready to use**:

- ✅ Quick Start Guide - Ready
- ✅ Architecture Strategy - Ready
- ✅ Migration Guide - Ready
- ✅ Ceph Deployment Guide - Ready

## 🔄 Maintenance

These documents should be updated when:
- New storage technologies are adopted
- Storage architecture changes
- New apps with different storage needs are added
- Lessons learned from production issues
- Backup/recovery procedures change

## 🌟 Best Practices

Captured throughout the documentation:
1. Use TrueNAS NFS for all persistent data
2. Keep storage architecture simple
3. Test backups regularly
4. Monitor storage health
5. Document recovery procedures
6. Start small, migrate incrementally
7. Validate before migrating next app

---

**Ready to get started?** → Read [STORAGE-QUICK-START.md](./STORAGE-QUICK-START.md) first!
