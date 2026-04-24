# Quick Start Guide

This guide will help you deploy your Solace Cache Helm chart.

## Prerequisites

1. **Build your Solace Cache container image** following the instructions at:
   https://github.com/SolaceLabs/PubSubCacheDocker

2. **Push the image to your registry:**
   ```bash
   docker tag solcache:latest your-registry.example.com/solace-cache:1.0.0
   docker push your-registry.example.com/solace-cache:1.0.0
   ```

3. **Have a running Solace broker** with cache configuration enabled

## Step 1: Configure Your Values

Edit `values.yaml` or create a custom values file:

```bash
cp values.yaml my-values.yaml
```

Update the following in `my-values.yaml`:

```yaml
image:
  repository: your-registry.example.com/solace-cache
  tag: "1.0.0"

solaceCache:
  broker:
    host: "tcp://your-broker.example.com:55555"
    vpn: "your-vpn-name"
    username: "cache-username"
    password: "cache-password"
```

## Step 2: Create Credentials Secret (Recommended for Production)

Instead of storing passwords in values.yaml:

```bash
kubectl create secret generic solace-cache-creds \
  --from-literal=username=cache-username \
  --from-literal=password=your-secure-password
```

Then update `my-values.yaml`:

```yaml
solaceCache:
  broker:
    existingSecret: "solace-cache-creds"
```

## Step 3: Install the Chart

```bash
# Dry run first to verify
helm install my-cache . -f my-values.yaml --dry-run --debug

# If everything looks good, install
helm install my-cache . -f my-values.yaml
```

## Step 4: Verify Deployment

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=solace-cache

# View logs
kubectl logs -l app.kubernetes.io/name=solace-cache -f

# Check if both pods are running on different nodes (for HA)
kubectl get pods -l app.kubernetes.io/name=solace-cache -o wide
```

Expected output:
```
NAME                            READY   STATUS    RESTARTS   AGE   NODE
my-cache-solace-cache-abc123    1/1     Running   0          2m    node-1
my-cache-solace-cache-def456    1/1     Running   0          2m    node-2
```

## Step 5: Configure on Solace Broker

After your cache instances are running:

1. Log into your Solace Manager
2. Navigate to the Distributed Cache configuration
3. Add your cache instances
4. Configure cache topics and settings

Refer to the [Solace documentation](https://docs.solace.com/Solace-PubSub-Cache/Configuring-and-Managing-PubSub-Cache.htm) for details.

## Common Commands

```bash
# View chart values
helm get values my-cache

# Upgrade after making changes
helm upgrade my-cache . -f my-values.yaml

# Rollback if needed
helm rollback my-cache

# Uninstall
helm uninstall my-cache

# View rendered templates
helm template my-cache . -f my-values.yaml
```

## Testing Locally with Minikube

If you want to test locally:

```bash
# Start minikube with 2 nodes
minikube start --nodes 2

# Load your image into minikube
minikube image load your-registry.example.com/solace-cache:1.0.0

# Install the chart
helm install test-cache . -f my-values.yaml

# Check the deployment
kubectl get all
```

## Troubleshooting

### Pods stuck in ImagePullBackOff
- Verify image name and tag are correct
- Check imagePullSecrets if using private registry
- Ensure image is accessible from your cluster

### Pods CrashLoopBackOff
- Check logs: `kubectl logs <pod-name>`
- Verify broker connection details
- Ensure broker is reachable from the cluster
- Check if VPN and credentials are correct

### Health checks failing
- Adjust probe configuration in values.yaml
- Verify the process name matches what's used in health checks
- Consider adding custom health check endpoints

## Next Steps

Once your cache is running:

1. Monitor resource usage and adjust requests/limits
2. Set up monitoring and alerting
3. Test failover scenarios
4. Review and adjust the PodDisruptionBudget based on your SLA
5. Consider using StatefulSet if stable network identities are needed

For production deployments, see `values-production.yaml` for recommended settings.
