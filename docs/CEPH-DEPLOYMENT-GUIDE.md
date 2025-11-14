# Ceph Deployment Guide for Proxmox Cluster

This guide covers deploying Ceph distributed storage across your 3-node Proxmox cluster for VM high availability and optional K3s integration.

## ⚠️ Important Decision Point

**Before You Start**: Ceph deployment is **OPTIONAL** and only recommended if you need:

- ✅ **VM live migration** between Proxmox nodes
- ✅ **High availability** for K3s control plane VMs
- ✅ **Redundant storage** for critical cluster infrastructure

**DO NOT use Ceph for:**
- ❌ Application data storage (use TrueNAS NFS instead)
- ❌ Media files (use TrueNAS NFS instead)
- ❌ Large datasets (TrueNAS is simpler and more efficient)

**Key Insight**: Your TrueNAS already provides reliable, high-performance storage. Ceph adds complexity and should only be used for Proxmox-specific features like VM live migration.

## Prerequisites

### Hardware Requirements

**Minimum per node:**
- 3+ nodes (you have 3 ✓)
- 2+ cores per node
- 4GB+ RAM per node (8GB+ recommended)
- Separate disk for Ceph OSDs (not the OS disk)
- 10Gbps+ networking recommended (you have 5Gbps on 2 nodes)

**Your Setup:**
- Main Station PC (RTX 3090)
- XPS 15 (5Gb networking)
- Razer 15 (5Gb networking)

### Network Requirements

**Recommended**: Separate networks for Ceph
- Public network: Client access (10.10.0.0/16)
- Cluster network: Ceph replication (dedicated VLAN recommended)

**Your Setup**: 5Gb backbone between XPS and Razer, 1Gb on main PC (verify)

### Storage Requirements

**Per node**: At least one dedicated disk for Ceph OSDs
- SSD/NVMe highly recommended for performance
- Separate from OS disk
- Raw device (no partitions or file systems)

## Architecture Options

### Option 1: Proxmox Integrated Ceph (Recommended)

Use Proxmox's built-in Ceph management for simplicity.

**Pros:**
- Integrated into Proxmox web UI
- Easier management and monitoring
- Automatic integration with Proxmox storage
- Good for VM storage only

**Cons:**
- Less flexible than standalone Ceph
- Tied to Proxmox version
- Not easily accessible from K3s

**Best for**: VM high availability without K3s integration

### Option 2: Rook-Ceph in Kubernetes

Deploy Ceph inside your K3s cluster using Rook operator.

**Pros:**
- Kubernetes-native management
- Direct access from K3s pods
- GitOps-friendly configuration
- CSI driver built-in

**Cons:**
- More complex setup
- Consumes cluster resources
- Chicken-and-egg: Ceph running on VMs using Ceph storage

**Best for**: K3s-native workloads needing distributed storage

### Option 3: Standalone Ceph Cluster

Deploy Ceph separately from both Proxmox and K3s.

**Pros:**
- Most flexible
- Can serve both Proxmox and K3s
- Best performance
- Cleanest separation of concerns

**Cons:**
- Most complex to manage
- Requires separate management interface
- Additional operational overhead

**Best for**: Advanced users, production environments

## Recommended Approach

For your homelab, I recommend **Option 1: Proxmox Integrated Ceph** for these reasons:

1. Simplest to set up and manage
2. You only need it for VM live migration
3. Application data is already on TrueNAS
4. Reduces operational complexity

## Deployment: Proxmox Integrated Ceph

### Step 1: Prepare Storage Devices

On each Proxmox node, identify disks for Ceph OSDs:

```bash
# List all disks
lsblk

# Check disk health
smartctl -a /dev/sdX

# Wipe disk if reusing (DESTRUCTIVE!)
sgdisk --zap-all /dev/sdX
dd if=/dev/zero of=/dev/sdX bs=1M count=100
```

**Example configuration:**
- **Main PC**: Use NVMe or SSD (e.g., `/dev/nvme1n1`)
- **XPS 15**: Use SSD (e.g., `/dev/sdb`)
- **Razer 15**: Use SSD (e.g., `/dev/sdc`)

### Step 2: Install Ceph via Proxmox UI

On your primary Proxmox node's web interface:

1. **Navigate to Datacenter → Ceph**
2. Click **Install Ceph**
3. Select:
   - **Version**: Quincy or Reef (latest stable)
   - **Repository**: No-subscription (for homelab)
4. Click **Start Installation**
5. Repeat on all 3 nodes

### Step 3: Create Ceph Cluster

On the **first node only** (e.g., Main PC):

1. **Datacenter → Ceph → Configuration**
2. Click **Create Configuration**
3. Configure:
   - **Public Network**: `10.10.0.0/16` (your main network)
   - **Cluster Network**: `10.10.0.0/16` (or dedicated if available)
   - **Number of replicas**: 3 (use all nodes)
   - **Minimum replicas**: 2 (survive 1 node failure)
4. Click **Create**

### Step 4: Create Ceph Monitors

Monitors (MONs) maintain cluster state. Create one per node:

1. **Node → Ceph → Monitor**
2. Click **Create Monitor**
3. Select network interface
4. Click **Create**
5. Repeat on other 2 nodes

Verify all monitors are running:
```bash
pveceph mon list
ceph -s
```

### Step 5: Create Ceph Managers

Managers handle cluster management APIs:

1. **Node → Ceph → Manager**
2. Click **Create Manager**
3. Repeat on at least 2 nodes (for HA)

Verify:
```bash
pveceph mgr list
```

### Step 6: Create OSDs

Object Storage Daemons (OSDs) store actual data:

On each node:

1. **Node → Ceph → OSD**
2. Click **Create OSD**
3. Select disk (e.g., `/dev/nvme1n1`)
4. Choose:
   - **DB/WAL disk**: Same disk (or separate SSD for performance)
   - **OSD class**: Default
5. Click **Create**

Verify OSDs:
```bash
pveceph osd tree
ceph osd status
```

You should see 3 OSDs (one per node) with status `up` and `in`.

### Step 7: Create Ceph Pools

Pools define replication and placement rules:

```bash
# For VM storage (default pool)
pveceph pool create vm-storage
pveceph pool set vm-storage size 3
pveceph pool set vm-storage min_size 2

# For K3s etcd backups (optional)
pveceph pool create k3s-etcd
pveceph pool set k3s-etcd size 3
pveceph pool set k3s-etcd min_size 2
```

Or via UI:
1. **Datacenter → Ceph → Pools**
2. Click **Create**
3. Configure pool settings

### Step 8: Create Proxmox Storage

Add Ceph as Proxmox storage backend:

1. **Datacenter → Storage**
2. Click **Add → RBD**
3. Configure:
   - **ID**: `ceph-vm-storage`
   - **Pool**: `vm-storage`
   - **Content**: Disk image, Container
   - **Nodes**: Select all 3 nodes
4. Click **Add**

### Step 9: Test Ceph Cluster

```bash
# Check cluster health
ceph -s

# Expected output:
#   health: HEALTH_OK
#   mon: 3 daemons
#   mgr: 2 daemons, 1 active
#   osd: 3 osds: 3 up, 3 in

# Test write performance
rados bench -p vm-storage 10 write --no-cleanup

# Test read performance
rados bench -p vm-storage 10 seq

# Clean up test data
rados -p vm-storage cleanup
```

### Step 10: Migrate K3s Control Plane VM to Ceph

Now you can migrate your K3s control plane VM to use Ceph storage:

1. **Backup VM first!**
   ```bash
   vzdump <VMID> --mode snapshot --storage local
   ```

2. **Migrate VM disk to Ceph:**
   ```bash
   # GUI: VM → Hardware → Hard Disk → Move Disk
   # Select target: ceph-vm-storage
   
   # Or via CLI:
   qm move-disk <VMID> scsi0 ceph-vm-storage --delete 1
   ```

3. **Test VM migration:**
   ```bash
   # Migrate VM to another node (should be live)
   qm migrate <VMID> <target-node> --online
   ```

## Monitoring and Maintenance

### Daily Checks

```bash
# Cluster health
ceph -s

# Check for warnings
ceph health detail

# Monitor OSD status
ceph osd status

# Check pool usage
ceph df
```

### Grafana Dashboard

Proxmox provides built-in Ceph monitoring:

1. **Datacenter → Ceph → Monitor**
2. View graphs for:
   - IOPS
   - Throughput
   - Latency
   - Pool usage

### Prometheus Integration

Export Ceph metrics to your existing Prometheus:

```bash
# Enable Ceph Prometheus module
ceph mgr module enable prometheus

# Access metrics
curl http://<any-node>:9283/metrics
```

Add to Prometheus scrape config:
```yaml
# infrastructure/configs/staging/prometheus/prometheus.yaml
scrape_configs:
  - job_name: 'ceph'
    static_configs:
      - targets:
          - '<node1-ip>:9283'
          - '<node2-ip>:9283'
          - '<node3-ip>:9283'
```

## Troubleshooting

### Cluster Not Healthy

```bash
# Check detailed health
ceph health detail

# Common issues:
# - Clock skew: Sync NTP on all nodes
# - OSDs down: Check disk health
# - PGs inconsistent: Run scrub
```

### OSD Won't Start

```bash
# Check OSD logs
journalctl -u ceph-osd@<osd-id>

# Check disk
smartctl -a /dev/sdX

# Remove and recreate OSD if disk failed
ceph osd out <osd-id>
ceph osd purge <osd-id> --yes-i-really-mean-it
```

### Slow Performance

```bash
# Check network
iperf3 -s  # on node 1
iperf3 -c <node1-ip>  # on node 2

# Check OSD latency
ceph osd perf

# Check pool replication
ceph osd pool ls detail
```

### Split Brain / MON Issues

```bash
# Check MON status
ceph mon stat

# Restart MON
systemctl restart ceph-mon@<hostname>

# Worst case: Remove and re-add MON
ceph mon remove <hostname>
pveceph mon create --monid <hostname>
```

## Performance Tuning

### Network Optimization

```bash
# Enable jumbo frames (if supported)
# On all nodes:
ip link set dev <interface> mtu 9000

# Verify
ip link show <interface>
```

### Ceph Configuration Tuning

```bash
# Increase OSD threads (more CPU, better performance)
ceph config set osd osd_op_threads 8

# Adjust recovery throttle (faster recovery, more IO impact)
ceph config set osd osd_recovery_max_active 6

# BlueStore cache (more RAM, better performance)
ceph config set osd bluestore_cache_size_hdd 4294967296  # 4GB
ceph config set osd bluestore_cache_size_ssd 8589934592  # 8GB
```

### Pool Optimization

```bash
# Enable fast read for RBD
ceph osd pool set vm-storage fast_read true

# Set PG autoscaler
ceph osd pool set vm-storage pg_autoscale_mode on
```

## Backup and Disaster Recovery

### Backup Strategy

**Ceph Cluster Backup:**
- Export cluster configuration
- Backup monitor databases
- Use Proxmox backup for VM disks

```bash
# Backup Ceph config
ceph config dump > /root/ceph-config-backup.txt

# Backup MON keyring
cp /etc/ceph/ceph.client.admin.keyring /root/
```

**VM Backup on Ceph:**
- Use Proxmox Backup Server (PBS)
- Schedule regular VM snapshots
- Replicate to TrueNAS or off-site

### Disaster Recovery Scenarios

| Scenario | Recovery |
|----------|----------|
| 1 OSD fails | Automatic: Ceph replicates data |
| 1 node fails | Automatic: VMs migrate, Ceph rebalances |
| 2 nodes fail | Degraded: Cluster read-only, data intact |
| Complete cluster loss | Rebuild cluster, restore from backup |

### Testing Failover

```bash
# Test OSD failure
systemctl stop ceph-osd@0

# Verify cluster still healthy (with warnings)
ceph -s

# Ceph should rebalance automatically
watch ceph -s

# Bring OSD back
systemctl start ceph-osd@0
```

## Security Considerations

### CephX Authentication

Enabled by default. Verify:

```bash
ceph auth list
```

### Network Isolation

**Recommended**: Separate cluster network
- Client/public: K3s, Proxmox access
- Cluster: Ceph replication only

Configure in `/etc/ceph/ceph.conf`:
```ini
[global]
public network = 10.10.0.0/16
cluster network = 10.20.0.0/16
```

### Encryption at Rest

**Optional**: Enable encryption on OSDs

```bash
# When creating OSD, use:
pveceph osd create /dev/sdX --encrypted 1
```

## Cost-Benefit Analysis

### Benefits of Ceph

✅ **VM live migration** between nodes
✅ **HA for K3s control plane** VMs
✅ **No single point of failure** for VM storage
✅ **Automatic rebalancing** on node failure

### Costs of Ceph

❌ **Complexity**: More components to manage
❌ **Resources**: Consumes CPU, RAM, disk, network
❌ **Performance overhead**: 3x replication
❌ **Operational burden**: Monitoring, maintenance, troubleshooting

### Alternative: TrueNAS Only

If you decide Ceph is overkill:

**Alternative approach:**
1. Keep VMs on local storage (no live migration)
2. Use Proxmox HA with VM restart (not live migration)
3. Store all VM data on TrueNAS NFS (stateless VMs)
4. Use GitOps to rebuild VMs quickly

**Trade-offs:**
- ✅ Much simpler architecture
- ✅ Fewer moving parts
- ❌ No live migration (VMs restart on node failure)
- ❌ Brief downtime during node maintenance

## Recommendations

### When to Deploy Ceph

✅ **Deploy Ceph if:**
- You need VM live migration
- Downtime for VM restarts is unacceptable
- You enjoy managing complex storage systems
- You have spare resources (disk, RAM, CPU)

### When to Skip Ceph

❌ **Skip Ceph if:**
- Application data is already on TrueNAS
- Brief VM restart downtime is acceptable
- You prefer simplicity over features
- Resources are limited

### My Recommendation for Your Setup

**Phase 1 (Now)**: Skip Ceph initially
- Migrate apps to TrueNAS NFS (Phase 1 complete)
- Keep K3s control plane VMs on local storage
- Use GitOps for fast rebuild

**Phase 2 (Later)**: Evaluate Ceph need
- After running TrueNAS-backed apps for a few months
- If you find VM restart downtime painful
- If you want to experiment with distributed storage

**Phase 3 (Future)**: Deploy Ceph if needed
- Follow this guide to set up Ceph
- Migrate only control plane VMs to Ceph
- Keep application data on TrueNAS

## Conclusion

Ceph is a powerful distributed storage system that can provide VM high availability for your Proxmox cluster. However, it adds significant complexity and should only be deployed if you specifically need its features.

For your homelab:
- **Primary storage**: TrueNAS NFS (simpler, reliable, fast)
- **VM storage**: Ceph (optional, only if you need live migration)
- **Application data**: TrueNAS (not Ceph)

**Remember**: The best storage architecture is the simplest one that meets your requirements. Start with TrueNAS, add Ceph only if you need it.

## Next Steps

**If deploying Ceph:**
1. Prepare dedicated disks on each Proxmox node
2. Follow deployment steps in this guide
3. Test thoroughly before migrating production VMs
4. Set up monitoring and alerting
5. Document your specific configuration

**If skipping Ceph:**
1. Continue with TrueNAS NFS migration (Phase 1)
2. Implement backup strategy for VMs
3. Use Proxmox HA with VM restart
4. Revisit Ceph decision in 3-6 months

## Additional Resources

- [Proxmox Ceph Documentation](https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster)
- [Ceph Documentation](https://docs.ceph.com/)
- [Rook-Ceph Kubernetes Operator](https://rook.io/docs/rook/latest/)
- [Ceph Performance Tuning](https://docs.ceph.com/en/latest/rados/configuration/bluestore-config-ref/)
