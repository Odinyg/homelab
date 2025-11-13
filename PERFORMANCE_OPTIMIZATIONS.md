# Performance Optimizations

This document describes the performance optimizations implemented in the homelab Kubernetes cluster and provides recommendations for monitoring and further improvements.

## Implemented Optimizations

### 1. Resource Management

**Problem:** Deployments without resource requests/limits can cause:
- Resource starvation of other pods
- Unpredictable performance
- OOM (Out of Memory) kills
- Poor scheduling decisions

**Solution:** Added appropriate resource requests and limits to all application deployments:

| Application | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-------------|-------------|-----------|----------------|--------------|
| open-webui | 100m | 500m | 256Mi | 1Gi |
| ollama | 500m | 2000m | 2Gi | 8Gi |
| audiobookshelf | 100m | 1000m | 256Mi | 1Gi |
| linkding | 50m | 500m | 128Mi | 512Mi |
| karakeep-web | 100m | 1000m | 256Mi | 1Gi |
| meilisearch | 100m | 300m | 128Mi | 256Mi |

**Benefits:**
- Guaranteed minimum resources for each application
- Prevention of resource hogging
- Better pod scheduling and distribution
- Predictable performance under load

### 2. Health Probes

**Problem:** Without health probes:
- Failed containers continue receiving traffic
- Slower failure detection
- Manual intervention required for recovery
- Increased downtime

**Solution:** Added liveness and readiness probes to all deployments:

**Liveness Probes** (restart unhealthy containers):
- Initial delay: 30-60 seconds (allows startup time)
- Period: 10-30 seconds
- Timeout: 5-10 seconds
- Failure threshold: 3

**Readiness Probes** (control traffic routing):
- Initial delay: 10-30 seconds
- Period: 5-10 seconds
- Timeout: 3-5 seconds
- Failure threshold: 3

**Probe Endpoints Used:**
- ollama: `/api/version` (liveness), `/api/tags` (readiness) - documented Ollama API endpoints
- open-webui: `/` (root path - standard for web apps)
- audiobookshelf: `/ping` (documented health check)
- linkding: `/` (root path - reliable connectivity check)
- karakeep-web: `/` (root path - Next.js app)
- meilisearch: `/health` (standard health endpoint)

**Benefits:**
- Automatic detection and restart of failed containers
- Prevention of traffic to unhealthy pods
- Faster recovery from transient failures
- Reduced manual intervention

### 3. FluxCD Reconciliation Optimization

**Problem:** Excessive reconciliation intervals cause:
- Unnecessary API calls to Git and Helm repositories
- Increased cluster CPU/memory usage
- Network bandwidth waste
- Higher cloud costs (if applicable)

**Original Configuration:**
- Longhorn HelmRepository: 1m (checks every minute)
- Longhorn HelmRelease: 1m (reconciles every minute)
- cert-manager HelmRepository: 5m
- GitRepository flux-system: 1m

**Optimized Configuration:**
- Longhorn HelmRepository: 24h (Helm charts rarely change)
- Longhorn HelmRelease: 30m with 12h chart interval
- cert-manager HelmRepository: 24h
- GitRepository: Managed by Flux (remains at 1m for rapid deployments)

**Impact:**
- **288x reduction** in Helm repository checks (1440 checks/day → 5 checks/day)
- **30x reduction** in Longhorn reconciliation (1440/day → 48/day)
- Significantly reduced API calls and cluster load
- Maintained timely updates while reducing overhead

### 4. Storage Configuration

**Problem:** linkding PVC was missing explicit storageClassName, which could lead to:
- Unpredictable provisioner selection
- Inconsistent storage performance
- Difficult troubleshooting

**Solution:** 
- Added explicit `storageClassName: local-path` to linkding PVC
- Ensures consistent storage provisioning

## Monitoring Recommendations

### Resource Usage Monitoring

Use these Prometheus queries in Grafana to monitor resource usage:

```promql
# CPU usage by pod
rate(container_cpu_usage_seconds_total[5m])

# Memory usage by pod
container_memory_working_set_bytes

# Pod CPU throttling (indicates CPU limits are too low)
rate(container_cpu_cfs_throttled_seconds_total[5m])

# Pods close to memory limits
container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.8
```

### Health Probe Monitoring

```promql
# Liveness probe failures
rate(prober_probe_total{probe_type="Liveness",result="failed"}[5m])

# Readiness probe failures
rate(prober_probe_total{probe_type="Readiness",result="failed"}[5m])

# Container restarts (may indicate probe issues)
rate(kube_pod_container_status_restarts_total[15m]) > 0
```

### FluxCD Monitoring

```promql
# Reconciliation duration
gotk_reconcile_duration_seconds_bucket

# Reconciliation failures
gotk_reconcile_condition{type="Ready",status="False"}
```

## Additional Optimization Opportunities

### 1. Startup Probes for Slow-Starting Applications

Consider adding startup probes for applications with long initialization times (e.g., ollama with large models):

```yaml
startupProbe:
  httpGet:
    path: /
    port: 11434
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes total
```

### 2. Horizontal Pod Autoscaling (HPA)

For applications with variable load, consider adding HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: open-webui
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: open-webui
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 3. Pod Disruption Budgets (PDB)

For critical applications, add PDBs to ensure availability during updates:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: linkding
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: linkding
```

### 4. Resource Quotas per Namespace

Consider adding resource quotas to prevent resource exhaustion:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

### 5. Network Policies

Add network policies to reduce unnecessary network traffic and improve security:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: karakeep-policy
spec:
  podSelector:
    matchLabels:
      app: karakeep-web
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: traefik
    ports:
    - protocol: TCP
      port: 3000
```

### 6. Node Affinity and Anti-Affinity

For better distribution and performance:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: audiobookshelf
        topologyKey: kubernetes.io/hostname
```

### 7. Priority Classes

Set priorities to ensure critical workloads are scheduled first:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
```

## Performance Testing

After implementing these optimizations, verify improvements using:

### 1. Resource Usage
```bash
# Check pod resource usage
kubectl top pods -A

# Check node resource usage
kubectl top nodes
```

### 2. Application Response Times
```bash
# Test response time for each app
time curl -s http://linkding.pytt.io/ > /dev/null
time curl -s http://gr.pytt.io > /dev/null
time curl -s http://ollama:11434/api/version > /dev/null
```

### 3. Flux Reconciliation
```bash
# Check Flux status
flux get all -A

# View reconciliation duration
flux get kustomizations --watch
```

### 4. Pod Stability
```bash
# Check for restarts (should be minimal)
kubectl get pods -A -o wide | grep -v "0/"

# Check events for issues
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

## Rollback Plan

If issues occur after these changes:

1. **Resource Issues:**
   ```bash
   # Increase limits if throttling occurs
   kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"limits":{"cpu":"1000m"}}}]}}}}'
   ```

2. **Probe Issues:**
   ```bash
   # Adjust probe timings if false positives occur
   kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","livenessProbe":{"initialDelaySeconds":60}}]}}}}'
   ```

3. **Flux Issues:**
   ```bash
   # Force immediate reconciliation if needed
   flux reconcile helmrelease longhorn-release --with-source
   ```

## Conclusion

These optimizations provide:

- **Better resource utilization** through explicit requests/limits
- **Improved reliability** through health probes and automatic recovery
- **Reduced cluster load** through optimized reconciliation intervals
- **More predictable behavior** through explicit configurations

Monitor the cluster after deployment to ensure all applications function correctly with the new configurations. Adjust resource limits based on actual usage patterns observed in Prometheus/Grafana.
