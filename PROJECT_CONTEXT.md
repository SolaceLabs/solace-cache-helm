# Solace Cache Helm Chart - Implementation Context

**Chart Version:** 0.1.0  
**App Version:** 1.0.11  
**Created:** April 2026  
**Purpose:** Production-ready Helm chart for deploying Solace Cache instances (Linux C API application)

---

## Architecture Overview

### Design Philosophy: "Pets, Not Cattle"
- These cache instances are **stateful "pets"** that must maintain maximum uptime
- Each pod has a **unique identity** and specific configuration
- Focus on **HA protection** and graceful handling of disruptions
- Use StatefulSet for stable pod identities and ordered deployment

### Key Components
1. **StatefulSet** - Main workload (not Deployment)
2. **Headless Service** - DNS resolution for pods (no inbound ports needed)
3. **ConfigMap** - Template config with placeholders for per-pod substitution
4. **Secret** - Broker credentials (username/password)
5. **PodDisruptionBudget** - Prevents voluntary disruptions from affecting availability
6. **Init Container** - Generates unique config per pod before main container starts

---

## Critical Implementation Details

### 1. Per-Pod Configuration Strategy
**Problem:** Each cache instance needs a unique `CACHE_INSTANCE_NAME` that matches the broker's distributed cache configuration.

**Solution:** Init container extracts pod ordinal and selects from `instanceNames` array:
```bash
# In init container
ORDINAL=${HOSTNAME##*-}
INSTANCE_NAME=$(yq eval ".solaceCache.instanceNames[${ORDINAL}]" /etc/config-values/values.yaml)
sed "s/__CACHE_INSTANCE_NAME__/${INSTANCE_NAME}/g" /etc/config-template/config.txt > /config/config.txt
```

### 2. Signal Handling for Fast Shutdown
**Problem:** Container was taking 30 seconds to shut down (SIGTERM timeout).

**Solution:** Use `exec` wrapper so SolaceCache becomes PID 1:
```yaml
command: ["/bin/sh", "-c"]
args:
  - |
    exec /home/solace/SolaceCache/bin/SolaceCache
```
**Result:** Shutdown time reduced from 30s to ~2s.

### 3. Config Change Detection
**Problem:** Updating ConfigMap didn't trigger pod restart.

**Solution:** Add checksum annotation to pod template:
```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
**Result:** Config changes now trigger rolling restart automatically.

### 4. Process Name for Health Checks
**Critical Detail:** The process name is `SolaceCache` with capital S and C:
```yaml
livenessProbe:
  exec:
    command: ["pgrep", "-f", "SolaceCache"]
```
**NOT:** "solcache" or "solaceCache" - must match exactly.

---

## File Structure

### Configuration Files
- **values.yaml** - Base configuration with sensible defaults
  - 2 replicas (HA by default)
  - INFO log levels
  - PodDisruptionBudget enabled
  - Preferred pod anti-affinity
  
- **values-standalone.yaml** - Single instance for dev/UAT
  - 1 replica but with PDB protection
  - DEBUG log levels
  - Higher probe failure thresholds
  
- **values-prod-ha.yaml** - Production overrides only
  - Custom registry and image pull secrets
  - Higher resource limits (4Gi/4000m)
  - SSL enabled for broker connection
  - Required (strict) pod anti-affinity

### Templates
- **statefulset.yaml** - Main workload with init container
- **service.yaml** - Headless service (clusterIP: None)
- **configmap.yaml** - Config template with placeholders
- **secret.yaml** - Broker credentials (base64 encoded)
- **poddisruptionbudget.yaml** - HA protection (minAvailable: 1)
- **serviceaccount.yaml** - Optional RBAC
- **tests/test-connection.yaml** - Helm test verifying instances running

### Documentation
- **README.md** - Complete usage guide
- **QUICKSTART.md** - Fast deployment instructions
- **NOTES.txt** - Post-install instructions displayed to user
- **PROJECT_CONTEXT.md** - This file

---

## Configuration Parameters

### Key Settings in values.yaml

#### Instance Identity
```yaml
solaceCache:
  instanceNames:
    - "cache-instance-0"  # Must match broker config
    - "cache-instance-1"
  distributedCacheName: "my-distributed-cache"
```

#### Broker Connection
```yaml
  broker:
    host: "tcp://solace-broker:55555"
    vpn: "default"
    username: "cache-user"      # Or use existingSecret
    password: "cache-password"
    existingSecret: ""          # Recommended for production
```

#### Logging
```yaml
  settings:
    sdkLogLevel: "INFO"    # Solace API logging
    cacheLogLevel: "INFO"  # Cache instance logging
```

#### HA Protection
```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1  # At least 1 pod must remain during disruptions
```

---

## Deployment Patterns

### Development/UAT (Single Instance with Protection)
```bash
helm install my-cache . -f values-standalone.yaml
```
- 1 replica with PDB enabled (maximum uptime)
- DEBUG logging for troubleshooting
- Suitable for non-production environments

### Production HA
```bash
helm install prod-cache . -f values-prod-ha.yaml
```
- 2 replicas on separate nodes (required anti-affinity)
- SSL-enabled broker connection
- Higher resource allocations
- Secrets-based credentials

### Upgrading Configuration
```bash
helm upgrade my-cache . -f my-values.yaml
```
- Config changes trigger rolling restart (via checksum annotation)
- PDB ensures at least 1 pod remains available during upgrade
- StatefulSet updates pods in reverse ordinal order (1, then 0)

---

## Testing and Validation

### Pre-Install Validation
```bash
helm lint .
helm template test . --debug
```

### Post-Install Verification
```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=solace-cache

# View logs
kubectl logs -f solace-cache-0
kubectl logs -f solace-cache-1

# Run Helm test
helm test my-cache

# Check config generated correctly
kubectl exec solace-cache-0 -- cat /config/config.txt
```

### Health Check
```bash
kubectl exec solace-cache-0 -- pgrep -f SolaceCache
```
Should return PID. If empty, container is not running correctly.

---

## Known Issues and Gotchas

### 1. Process Name Must Be Exact
- Health probes use `pgrep -f SolaceCache`
- Must match capital S and C
- Tests use same pattern

### 2. Init Container Requires yq
- Base image must include `yq` for YAML parsing
- Init container extracts pod ordinal and selects instance name
- Alternative: Use shell-based parsing if yq unavailable

### 3. PDB and Single Replica
- PDB with minAvailable: 1 on single replica prevents all voluntary disruptions
- This is intentional for "pets" philosophy
- Node drains will be blocked until PDB is deleted or pod is force-evicted

### 4. Image Repository Placeholder
- Default `values.yaml` has placeholder: `your-registry/solace-cache`
- Must be updated before deployment
- Production values already override this

### 5. StatefulSet Update Strategy
- Uses `RollingUpdate` with `partition: 0`
- Updates pods in reverse order (highest ordinal first)
- Allows staged rollouts by increasing partition value

---

## Future Enhancements

### Potential Improvements
1. **Monitoring** - Add Prometheus metrics if SolaceCache exposes them
2. **Backup Strategy** - Document or automate cache state backup
3. **Multi-Region** - Add topology spread constraints for zone awareness
4. **Readiness Gates** - External validation before marking pod ready
5. **Config Validation** - Pre-flight checks in init container
6. **Syslog Integration** - Enable syslog forwarding for centralized logging

### Not Implemented (By Design)
- **HPA** - Autoscaling disabled; replicas should be manually controlled
- **Ingress** - No inbound traffic; cache connects to broker only
- **Persistence** - Uses emptyDir; cache state is ephemeral per pod lifecycle

---

## Troubleshooting Guide

### Pods Not Starting
```bash
kubectl describe pod solace-cache-0
kubectl logs solace-cache-0 -c init-config  # Check init container
```
- Verify instanceNames array has enough entries for replica count
- Check secret exists if using existingSecret
- Ensure image is pullable from registry

### Slow Shutdown
- Verify exec wrapper is present in StatefulSet command
- Check if process is running as PID 1: `kubectl exec pod -- ps aux`

### Config Not Updating
- Check if checksum annotation is in statefulset.yaml pod template
- Verify ConfigMap was actually changed
- Force restart: `kubectl rollout restart statefulset/solace-cache`

### Health Probes Failing
- Verify process name: `kubectl exec pod -- pgrep -f SolaceCache`
- Check startup time; may need longer initialDelaySeconds
- Review logs for crash loops

### PDB Blocking Node Drain
```bash
kubectl get pdb
kubectl describe pdb solace-cache
```
- Expected behavior for "pets" with single replica
- Temporarily disable: `kubectl delete pdb solace-cache`
- Re-enable after drain: `helm upgrade --reuse-values`

---

## Packaging and Distribution

### Create Chart Archive
```bash
cd /path/to/solace-cache
helm package .
```
Produces: `solace-cache-0.1.0.tgz`

### Install from Package
```bash
helm install my-cache solace-cache-0.1.0.tgz -f my-values.yaml
```

### Version Management
Update Chart.yaml version field, then repackage:
```yaml
version: 0.2.0  # Chart version
appVersion: 1.0.12  # Application version
```

---

## Repository Information
- **GitHub:** Posted by user (April 2026)
- **Chart Type:** Application
- **License:** Not specified
- **Maintainer:** Add to Chart.yaml if needed

---

## Summary

This Helm chart implements a production-ready deployment for Solace Cache with:
- ✅ Unique per-pod configuration via init container
- ✅ Fast graceful shutdown via exec wrapper
- ✅ Automatic rolling restart on config changes
- ✅ HA protection with PodDisruptionBudget
- ✅ Node distribution via pod anti-affinity
- ✅ Multiple deployment profiles (dev/prod)
- ✅ Comprehensive documentation and testing

**Philosophy:** Treat cache instances as stateful "pets" requiring maximum uptime and careful handling during disruptions. The chart prioritizes availability and correctness over scalability.
