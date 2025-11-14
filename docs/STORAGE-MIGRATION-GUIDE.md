# Storage Migration Guide

This guide provides step-by-step instructions for migrating your K3s applications from local-path storage to TrueNAS NFS, implementing the architecture outlined in [STORAGE-ARCHITECTURE.md](./STORAGE-ARCHITECTURE.md).

## Prerequisites

- [ ] TrueNAS Scale accessible at 10.10.10.20
- [ ] K3s cluster with kubectl access
- [ ] SSH/console access to TrueNAS
- [ ] Backup of all existing application data

## Phase 1: TrueNAS Setup

### Step 1: Create ZFS Datasets

On TrueNAS, create a dedicated dataset structure for Kubernetes applications:

```bash
# SSH to TrueNAS or use the shell
zfs create tank/k8s-apps                    # Main parent dataset
zfs create tank/k8s-apps/ollama             # AI/ML models
zfs create tank/k8s-apps/linkding           # Bookmarks
zfs create tank/k8s-apps/karakeep           # Karaoke data
zfs create tank/k8s-apps/databases          # Database files
zfs create tank/k8s-apps/backups            # Backup storage

# Set appropriate quotas (optional)
zfs set quota=1T tank/k8s-apps/ollama
zfs set quota=10G tank/k8s-apps/linkding
zfs set quota=50G tank/k8s-apps/karakeep
```

Alternatively, use TrueNAS Web UI:
1. Navigate to **Storage** → **Pools**
2. Click on your pool (e.g., `big`)
3. Click **Add Dataset**
4. Name: `k8s-apps`
5. Repeat for each subdirectory

### Step 2: Configure NFS Shares

#### Option A: Single Shared Export (Simpler)

Create one NFS export for the parent dataset:

**TrueNAS Web UI:**
1. Navigate to **Sharing** → **Unix Shares (NFS)**
2. Click **Add**
3. Configure:
   - **Path**: `/mnt/big/k8s-apps`
   - **Description**: Kubernetes application storage
   - **Maproot User**: root
   - **Maproot Group**: root
   - **Network**: `10.10.0.0/16` (adjust to your subnet)
   - **Security**: Enable NFSv4, disable NFSv3
4. Click **Submit**

#### Option B: Per-Application Exports (More Granular)

Create separate exports for each application:

```bash
# Using TrueNAS CLI or repeat in Web UI for each:
# /mnt/big/k8s-apps/ollama
# /mnt/big/k8s-apps/linkding
# /mnt/big/k8s-apps/karakeep
```

### Step 3: Configure ZFS Snapshots

Enable automated snapshots for data protection:

**TrueNAS Web UI:**
1. Navigate to **Tasks** → **Periodic Snapshot Tasks**
2. Click **Add**
3. Configure:
   - **Dataset**: `big/k8s-apps`
   - **Recursive**: Yes (include all child datasets)
   - **Schedule**:
     - Hourly: Keep 24
     - Daily: Keep 7
     - Weekly: Keep 4
     - Monthly: Keep 12
4. Click **Submit**

### Step 4: Test NFS Access

From your K3s node:

```bash
# Test mount manually
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs -o nfsvers=4.1 10.10.10.20:/mnt/big/k8s-apps /mnt/test-nfs

# Verify access
ls -la /mnt/test-nfs
touch /mnt/test-nfs/test-file
rm /mnt/test-nfs/test-file

# Unmount
sudo umount /mnt/test-nfs
```

## Phase 2: Deploy NFS CSI Provisioner

### Option A: Using NFS Subdir External Provisioner (Recommended)

This enables dynamic PV provisioning from NFS shares.

Create the following manifests:

```bash
mkdir -p infrastructure/controllers/base/nfs-provisioner
```

#### 1. Namespace

```yaml
# infrastructure/controllers/base/nfs-provisioner/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-provisioner
```

#### 2. RBAC

```yaml
# infrastructure/controllers/base/nfs-provisioner/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

#### 3. Deployment

```yaml
# infrastructure/controllers/base/nfs-provisioner/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs.csi.k8s.io
            - name: NFS_SERVER
              value: 10.10.10.20
            - name: NFS_PATH
              value: /mnt/big/k8s-apps
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.10.10.20
            path: /mnt/big/k8s-apps
```

#### 4. StorageClass

```yaml
# infrastructure/controllers/base/nfs-provisioner/storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: truenas-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"  # Don't make default yet
provisioner: nfs.csi.k8s.io
parameters:
  archiveOnDelete: "true"  # Rename PVs on delete instead of removing
mountOptions:
  - nfsvers=4.1
  - hard
  - timeo=600
  - retrans=2
  - rsize=1048576
  - wsize=1048576
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

#### 5. Kustomization

```yaml
# infrastructure/controllers/base/nfs-provisioner/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - rbac.yaml
  - deployment.yaml
  - storage-class.yaml
```

#### 6. Add to staging infrastructure

```yaml
# infrastructure/controllers/staging/kustomization.yaml
# Add nfs-provisioner to the resources list:
resources:
  - ../../base/nfs-provisioner
  # ... other resources
```

### Deploy the NFS Provisioner

```bash
# Build and preview
kubectl kustomize infrastructure/controllers/staging/

# Apply
kubectl apply -k infrastructure/controllers/staging/

# Verify deployment
kubectl get pods -n nfs-provisioner
kubectl get storageclass truenas-nfs
```

## Phase 3: Migrate Applications

### General Migration Process

For each application:

1. **Backup existing data**
2. **Scale down the application**
3. **Copy data to TrueNAS**
4. **Update manifests**
5. **Deploy with new storage**
6. **Verify functionality**
7. **Clean up old storage**

### Example: Migrate Ollama

#### Step 1: Backup Existing Data

```bash
# On K3s node, backup current ollama data
kubectl scale statefulset ollama -n ollama --replicas=0

# Create backup
sudo tar -czf /tmp/ollama-backup-$(date +%Y%m%d).tar.gz \
  /mnt/slow/ollama \
  /var/lib/rancher/k3s/storage/ollama
```

#### Step 2: Copy Data to TrueNAS

```bash
# Mount TrueNAS share
sudo mkdir -p /mnt/truenas
sudo mount -t nfs -o nfsvers=4.1 10.10.10.20:/mnt/big/k8s-apps /mnt/truenas

# Create ollama directory structure
sudo mkdir -p /mnt/truenas/ollama/{models,config}

# Copy data
sudo rsync -avP /mnt/slow/ollama/ /mnt/truenas/ollama/models/
sudo rsync -avP /var/lib/rancher/k3s/storage/ollama/config/ /mnt/truenas/ollama/config/

# Verify copy
ls -la /mnt/truenas/ollama/

# Unmount
sudo umount /mnt/truenas
```

#### Step 3: Update Ollama Storage Manifests

Replace the existing `apps/base/ollama/storage.yaml`:

```yaml
# apps/base/ollama/storage.yaml
---
# Models PVC (large, persistent)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-models-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: truenas-nfs
  resources:
    requests:
      storage: 500Gi
---
# Config PVC (small, persistent)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-config-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: truenas-nfs
  resources:
    requests:
      storage: 10Gi
---
# Cache PVC (ephemeral, fast)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-cache-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
```

Update the StatefulSet to use new PVCs:

```yaml
# apps/base/ollama/statefulset.yaml
# Update volumeMounts section:
volumeMounts:
  - name: models
    mountPath: /root/.ollama/models
  - name: config
    mountPath: /root/.ollama/config
  - name: cache
    mountPath: /tmp/ollama-cache

# Update volumes section:
volumes:
  - name: models
    persistentVolumeClaim:
      claimName: ollama-models-pvc
  - name: config
    persistentVolumeClaim:
      claimName: ollama-config-pvc
  - name: cache
    persistentVolumeClaim:
      claimName: ollama-cache-pvc
```

#### Step 4: Deploy with New Storage

```bash
# Delete old PVs and PVCs (after backup!)
kubectl delete pvc ollama-pvc ollama-config-pvc -n ollama
kubectl delete pv ollama-pv ollama-config-pv open-webui-pv

# Apply new configuration
kubectl apply -k apps/staging/ollama/

# Verify PVCs are bound
kubectl get pvc -n ollama

# Check pod status
kubectl get pods -n ollama -w
```

#### Step 5: Manually Create PV Directories (First Time Only)

The NFS provisioner should create subdirectories automatically, but you may need to manually move data:

```bash
# On TrueNAS or via NFS mount
sudo mount -t nfs -o nfsvers=4.1 10.10.10.20:/mnt/big/k8s-apps /mnt/truenas

# Check created PVs
ls -la /mnt/truenas/

# Copy data to provisioned directory (if needed)
# The provisioner creates directories like: ollama-ollama-models-pvc-pvc-xxxxx
sudo cp -a /mnt/truenas/ollama/models/* /mnt/truenas/ollama-ollama-models-pvc-pvc-xxxxx/

sudo umount /mnt/truenas
```

#### Step 6: Verify Functionality

```bash
# Check ollama is running
kubectl get pods -n ollama

# Test ollama API
kubectl port-forward -n ollama svc/ollama 11434:11434

# In another terminal:
curl http://localhost:11434/api/tags

# Verify models are available
kubectl exec -it -n ollama deployment/ollama -- ls -la /root/.ollama/models
```

#### Step 7: Clean Up Old Storage

Once verified working for 24-48 hours:

```bash
# Remove old data from local node
sudo rm -rf /mnt/slow/ollama
sudo rm -rf /var/lib/rancher/k3s/storage/ollama

# Keep backup for another week before deleting
```

### Migrate Linkding

Linkding uses a single PVC. Process is similar but simpler:

```bash
# 1. Scale down
kubectl scale deployment linkding -n linkding --replicas=0

# 2. Backup
sudo tar -czf /tmp/linkding-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/rancher/k3s/storage/linkding

# 3. Copy to TrueNAS
sudo mount -t nfs 10.10.10.20:/mnt/big/k8s-apps /mnt/truenas
sudo mkdir -p /mnt/truenas/linkding
sudo rsync -avP /var/lib/rancher/k3s/storage/linkding/ /mnt/truenas/linkding/
sudo umount /mnt/truenas

# 4. Update storage.yaml
```

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
  storageClassName: truenas-nfs
  resources:
    requests:
      storage: 5Gi
```

```bash
# 5. Apply and verify
kubectl delete pvc linkding-data-pvc -n linkding
kubectl apply -k apps/staging/linkding/
kubectl get pods -n linkding -w
```

### Migrate Karakeep

Similar process:

```yaml
# apps/base/karakeep/storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: karakeep-data-pvc
  namespace: karakeep
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: truenas-nfs
  resources:
    requests:
      storage: 50Gi
```

### Audiobookshelf (Already on NFS)

Audiobookshelf already uses NFS for media. Only migrate config/metadata:

```yaml
# apps/base/audiobookshelf/storage.yaml
# Update config and metadata PVCs to use truenas-nfs instead of local-path
# Keep audiobooks on existing NFS mount
```

## Phase 4: Set TrueNAS as Default Storage Class

Once all applications are migrated and stable:

```bash
# Remove default annotation from local-path
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set truenas-nfs as default
kubectl patch storageclass truenas-nfs \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
```

## Phase 5: Configure Backups

### Setup Automated ZFS Snapshot Monitoring

Install TrueNAS Prometheus exporter (optional):

```yaml
# Add to monitoring stack to track ZFS health
```

### Setup Application-Level Backups

For databases and critical data:

```yaml
# Example: CronJob for PostgreSQL backup to S3
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h postgres -U postgres mydb | \
              gzip > /backup/mydb-$(date +%Y%m%d).sql.gz
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-pvc  # truenas-nfs
          restartPolicy: OnFailure
```

## Troubleshooting

### PVC Stuck in Pending

```bash
# Check provisioner logs
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner

# Check events
kubectl describe pvc <pvc-name> -n <namespace>

# Verify NFS mount on node
kubectl get nodes -o wide
ssh <node-ip>
sudo mount | grep nfs
```

### Permission Denied Errors

```bash
# Check NFS export permissions on TrueNAS
# Make sure maproot is set correctly

# On TrueNAS, check dataset permissions
ls -la /mnt/big/k8s-apps/

# May need to adjust ownership
chown -R root:root /mnt/big/k8s-apps/
chmod -R 755 /mnt/big/k8s-apps/
```

### Slow Performance

```bash
# Check NFS mount options
kubectl get sc truenas-nfs -o yaml

# Test NFS performance
dd if=/dev/zero of=/mnt/truenas/testfile bs=1M count=1000
rm /mnt/truenas/testfile

# Consider enabling jumbo frames if supported
```

### Pod Won't Start After Migration

```bash
# Check pod logs
kubectl logs -n <namespace> <pod-name>

# Check PVC status
kubectl get pvc -n <namespace>

# Check if data was copied correctly
kubectl exec -it -n <namespace> <pod-name> -- ls -la /path/to/mount
```

## Rollback Procedure

If migration fails:

```bash
# 1. Scale down new deployment
kubectl scale deployment <app> -n <namespace> --replicas=0

# 2. Delete new PVCs
kubectl delete pvc <pvc-names> -n <namespace>

# 3. Restore old manifests from Git
git checkout HEAD~1 -- apps/base/<app>/storage.yaml

# 4. Reapply old configuration
kubectl apply -k apps/staging/<app>/

# 5. Restore from backup if needed
sudo tar -xzf /tmp/<app>-backup-*.tar.gz -C /
```

## Validation Checklist

After migration:

- [ ] All applications running and accessible
- [ ] Data integrity verified (spot checks)
- [ ] Performance acceptable (no major degradation)
- [ ] Backups tested (restore test)
- [ ] Monitoring dashboards showing storage metrics
- [ ] Documentation updated
- [ ] Old data backed up before removal
- [ ] Team notified of storage changes

## Next Steps

1. Monitor applications for 1-2 weeks
2. Set up automated backup verification
3. Document application-specific restore procedures
4. Plan Phase 2: MinIO S3 deployment
5. Consider Phase 3: Ceph deployment (if needed)

## Additional Resources

- [TrueNAS NFS Sharing](https://www.truenas.com/docs/scale/scaletutorials/shares/nfs/)
- [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [ZFS Snapshot Management](https://openzfs.github.io/openzfs-docs/man/8/zfs-snapshot.8.html)
