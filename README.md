# Solace Cache Helm Chart

This Helm chart deploys Solace Cache instances on Kubernetes with high availability configuration.

## Overview

Solace Cache is a distributed cache solution that connects to a Solace message broker. This chart is designed to deploy cache instances as "pets" not "cattle" - meaning these are stateful services that should maintain consistent availability rather than be scaled elastically.

**Important:** This chart uses a StatefulSet (not a Deployment) to ensure each cache instance has a stable identity and can be configured with a unique cache instance name that matches the configuration on the Solace broker.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A running Solace broker configured for cache usage
- Container image for Solace Cache (see Building the Container Image below)

## Building the Container Image

Before deploying this chart, you need to build a Docker image for Solace Cache:

1. Follow the instructions at: https://github.com/SolaceLabs/PubSubCacheDocker
2. Download the required Solace Cache and C API packages
3. Build the Docker image
4. Push the image to your container registry

Update `values.yaml` with your image repository and tag:

```yaml
image:
  repository: your-registry/solace-cache
  tag: "1.0.0"
```

## Installation

### Basic Installation

**Important:** You must configure cache instance names that match what you've configured on your Solace broker.

```bash
helm install my-cache ./solace-cache \
  --set solaceCache.instanceNames[0]="cache-inst-1" \
  --set solaceCache.instanceNames[1]="cache-inst-2" \
  --set solaceCache.distributedCacheName="my-distributed-cache" \
  --set solaceCache.broker.host="tcp://my-broker.example.com:55555" \
  --set solaceCache.broker.vpn="my-vpn" \
  --set solaceCache.broker.username="cache-user" \
  --set solaceCache.broker.password="cache-password"
```

### Installation with Existing Secret

For production, use Kubernetes Secrets for credentials:

```bash
# Create a secret with credentials
kubectl create secret generic solace-cache-creds \
  --from-literal=username=cache-user \
  --from-literal=password=cache-password

# Install using the secret
helm install my-cache ./solace-cache \
  --set solaceCache.broker.host="tcp://my-broker.example.com:55555" \
  --set solaceCache.broker.vpn="my-vpn" \
  --set solaceCache.broker.existingSecret="solace-cache-creds"
```

### Installation with SSL/TLS

```bash
# Create a secret with the broker's CA certificate
kubectl create secret generic solace-broker-ca \
  --from-file=ca.crt=/path/to/ca-certificate.pem

# Install with SSL enabled
helm install my-cache ./solace-cache \
  --set solaceCache.broker.host="tcps://my-broker.example.com:55443" \
  --set solaceCache.broker.vpn="my-vpn" \
  --set solaceCache.broker.existingSecret="solace-cache-creds" \
  --set solaceCache.broker.ssl.enabled=true \
  --set solaceCache.broker.ssl.trustStoreSecret="solace-broker-ca"
```

## Configuration

### Key Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of cache instances | `2` |
| `image.repository` | Container image repository | `your-registry/solace-cache` |
| `image.tag` | Container image tag | `latest` |
| `solaceCache.instanceNames` | Array of cache instance names (must match broker config) | `["cache-instance-0", "cache-instance-1"]` |
| `solaceCache.distributedCacheName` | Distributed cache name on broker | `default-cache` |
| `solaceCache.broker.host` | Solace broker connection URL | `tcp://solace-broker.default.svc.cluster.local:55555` |
| `solaceCache.broker.vpn` | Message VPN name | `default` |
| `solaceCache.broker.username` | Client username | `cache` |
| `solaceCache.broker.password` | Client password (use existingSecret in production) | `cache` |
| `solaceCache.broker.existingSecret` | Use existing secret for credentials | `""` |
| `solaceCache.broker.ssl.enabled` | Enable SSL/TLS connection | `false` |
| `solaceCache.broker.ssl.trustStoreSecret` | Secret containing CA certificate | `""` |
| `solaceCache.settings.sdkLogLevel` | API/SDK log level | `INFO` |
| `solaceCache.settings.cacheLogLevel` | Cache instance log level | `INFO` |
| `podDisruptionBudget.enabled` | Enable PodDisruptionBudget | `true` |
| `podDisruptionBudget.minAvailable` | Minimum available pods | `1` |
| `resources.requests.memory` | Memory request | `512Mi` |
| `resources.requests.cpu` | CPU request | `500m` |
| `resources.limits.memory` | Memory limit | `2Gi` |
| `resources.limits.cpu` | CPU limit | `2000m` |

### High Availability Features

This chart includes several HA features:

1. **StatefulSet**: Uses StatefulSet for stable pod identities
2. **Dual Replicas**: Runs 2 cache instances by default
3. **Unique Instance Names**: Each pod gets a unique cache instance name that must match the broker configuration
4. **Pod Disruption Budget**: Ensures at least 1 instance is always available during disruptions
5. **Pod Anti-Affinity**: Spreads instances across different nodes when possible
6. **Resource Limits**: Sets appropriate resource requests and limits

### Cache Instance Names

**Critical Configuration:** Each cache instance must have a unique name that matches what you've configured on the Solace broker:

1. Configure cache instances on your Solace broker (e.g., "prod-cache-1", "prod-cache-2")
2. Set the same names in your Helm values:

```yaml
solaceCache:
  instanceNames:
    - "prod-cache-1"  # Matches broker config
    - "prod-cache-2"  # Matches broker config
  distributedCacheName: "prod-distributed-cache"
```

The chart uses an init container to generate a unique config file for each pod based on its ordinal index.

### Custom Configuration

If you need to provide a custom `config.txt` file instead of the generated one:

```yaml
solaceCache:
  customConfig: |
    SESSION_USERNAME myuser
    SESSION_PASSWORD mypass
    SESSION_HOST tcp://broker:55555
    SESSION_VPN_NAME myvpn
    LOG_LEVEL INFO
    LOG_TO_STDOUT 1
    # Add your custom configuration here
```

## Upgrading

```bash
helm upgrade my-cache ./solace-cache -f custom-values.yaml
```

## Uninstalling

```bash
helm uninstall my-cache
```

## Monitoring

To check the status of your cache instances:

```bash
# View pods
kubectl get pods -l app.kubernetes.io/name=solace-cache

# Check logs
kubectl logs -l app.kubernetes.io/name=solace-cache -f

# Describe a specific pod
kubectl describe pod <pod-name>
```

## Troubleshooting

### Cache instances not connecting to broker

1. Check the logs: `kubectl logs <pod-name>`
2. Verify broker connectivity: `kubectl exec <pod-name> -- ping <broker-host>`
3. Verify credentials and VPN configuration
4. Check if the broker has cache configured

### Pods not starting

1. Verify the image is available: `kubectl describe pod <pod-name>`
2. Check resource constraints: `kubectl describe node`
3. Review events: `kubectl get events --sort-by='.lastTimestamp'`

## Important Notes

- **These are "pets" not "cattle"**: Do not use autoscaling. Keep replica count fixed at 2 for production.
- **StatefulSet**: This chart uses StatefulSet for stable identities and unique per-pod configuration
- **Instance names must match broker config**: The cache instance names in Helm values must exactly match what you've configured on the Solace broker
- **Health checks**: The default health checks use `pgrep`. Adjust based on your actual health check mechanism.
- **Resource sizing**: Adjust memory and CPU based on your cache size and throughput requirements.
- **Number of replicas**: If you change `replicaCount`, ensure you provide the correct number of instance names in `instanceNames` array.

## Next Steps

After deploying the cache instances:

1. Verify they are running and connected to the broker
2. Add the cache instances to your Distributed Cache on the Solace message router
3. Refer to [Solace Cache documentation](https://docs.solace.com/Solace-PubSub-Cache/Configuring-and-Managing-PubSub-Cache.htm) for configuration and management

## License

This Helm chart is provided as-is. Refer to Solace licensing for the Solace Cache software.
