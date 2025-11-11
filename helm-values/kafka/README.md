# Kafka Cluster Configuration

This directory contains Kafka cluster configurations for different environments using Strimzi Kafka Operator with KRaft mode (no ZooKeeper).

## Files

- **kafka-cluster.yaml** - Production configuration (3 replicas, full HA)
- **kafka-cluster-dev.yaml** - Development configuration (1 replica, minimal resources)

## Quick Start

### Use Default (Production - 3 Replicas)

The default configuration is already set in `kafka-cluster.yaml` with 3 replicas.

```bash
kubectl apply -f argocd/applications/kafka-cluster.yaml
```

### Switch to Development (1 Replica)

To use the development configuration with a single node:

```bash
# Backup the current configuration
cp helm-values/kafka/kafka-cluster.yaml helm-values/kafka/kafka-cluster.yaml.backup

# Use dev configuration
cp helm-values/kafka/kafka-cluster-dev.yaml helm-values/kafka/kafka-cluster.yaml

# Commit and push
git add helm-values/kafka/kafka-cluster.yaml
git commit -m "Switch to dev Kafka configuration (1 replica)"
git push origin main

# ArgoCD will auto-sync, or manually sync:
kubectl patch application kafka-cluster -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

### Switch Back to Production (3 Replicas)

```bash
# Restore production configuration
cp helm-values/kafka/kafka-cluster.yaml.backup helm-values/kafka/kafka-cluster.yaml

# Or just edit and change replicas back to 3
# Commit and push
git add helm-values/kafka/kafka-cluster.yaml
git commit -m "Switch to production Kafka configuration (3 replicas)"
git push origin main
```

## Scaling

You can scale the Kafka cluster at any time by editing the configuration:

### Scale Up (e.g., 1 → 3 replicas)

```bash
# Edit the configuration
vim helm-values/kafka/kafka-cluster.yaml

# Change:
# spec.replicas: 1
# To:
# spec.replicas: 3

# Also update replication factors:
# offsets.topic.replication.factor: 3
# transaction.state.log.replication.factor: 3
# transaction.state.log.min.isr: 2
# default.replication.factor: 3
# min.insync.replicas: 2

# Commit and push
git add helm-values/kafka/kafka-cluster.yaml
git commit -m "Scale Kafka to 3 replicas"
git push origin main
```

### Scale Down (e.g., 3 → 1 replicas)

⚠️ **Warning**: Scaling down can result in data loss if replication factor is higher than the number of remaining brokers.

```bash
# First, ensure topics can handle fewer replicas
# Set replication factors to 1:
vim helm-values/kafka/kafka-cluster.yaml

# Change replication factors to 1:
# offsets.topic.replication.factor: 1
# transaction.state.log.replication.factor: 1
# transaction.state.log.min.isr: 1
# default.replication.factor: 1
# min.insync.replicas: 1

# Then reduce replicas:
# spec.replicas: 1

# Commit and push
git add helm-values/kafka/kafka-cluster.yaml
git commit -m "Scale Kafka down to 1 replica"
git push origin main
```

## Configuration Comparison

| Setting | Development (1 node) | Production (3 nodes) |
|---------|---------------------|----------------------|
| **Replicas** | 1 | 3 |
| **Replication Factor** | 1 | 3 |
| **Min ISR** | 1 | 2 |
| **Memory per Pod** | 1-2Gi | 1-2Gi |
| **CPU per Pod** | 250m-1000m | 250m-1000m |
| **Storage per Pod** | 10Gi | 10Gi |
| **Total Resources** | 1-2Gi RAM, 0.25-1 CPU | 3-6Gi RAM, 0.75-3 CPU |
| **High Availability** | ❌ No | ✅ Yes |
| **Data Replication** | ❌ No | ✅ Yes |
| **Use Case** | Dev/Test | Production |

## Resource Requirements

### Development (1 replica)
```
Minimum: 1Gi RAM, 250m CPU
Recommended: 2Gi RAM, 500m CPU
Storage: 10Gi
```

### Production (3 replicas)
```
Minimum: 3Gi RAM, 750m CPU
Recommended: 6Gi RAM, 1.5 CPU
Storage: 30Gi (10Gi × 3)
```

## Troubleshooting

### Pods Not Starting (Insufficient Resources)

If you see errors like "Insufficient memory" or "Insufficient cpu":

1. **Option A**: Use development configuration (1 replica)
2. **Option B**: Reduce resource requests in the configuration
3. **Option C**: Add more nodes to your cluster

### Pods Unreachable

If pods are unreachable:

1. Check Strimzi operator logs:
   ```bash
   kubectl logs -n strimzi-system deployment/strimzi-cluster-operator
   ```

2. Check Kafka pod logs:
   ```bash
   kubectl logs -n kafka kafka-cluster-dual-role-0
   ```

3. Check network connectivity:
   ```bash
   kubectl exec -n kafka kafka-cluster-dual-role-0 -- nc -zv kafka-cluster-dual-role-1.kafka-cluster-kafka-brokers.kafka.svc.cluster.local 9092
   ```

4. Verify storage:
   ```bash
   kubectl get pvc -n kafka
   ```

### Topics Not Replicating

If you scaled up but topics aren't replicating:

```bash
# Increase replication factor for existing topics
kubectl run kafka-admin -ti --image=quay.io/strimzi/kafka:latest-kafka-3.8.0 --rm=true --restart=Never -- \
  bin/kafka-reassign-partitions.sh --bootstrap-server kafka-cluster-kafka-bootstrap:9092 \
  --topics-to-move-json-file /tmp/topics.json --broker-list "0,1,2" --generate

# Or use Kafka UI (port-forward to port 8080)
```

## Monitoring

Kafka metrics are exposed via JMX Prometheus Exporter. To monitor:

```bash
# Port-forward to Kafka pod
kubectl port-forward -n kafka kafka-cluster-dual-role-0 9404:9404

# Access metrics
curl http://localhost:9404/metrics
```

## Support

- [Strimzi Documentation](https://strimzi.io/documentation/)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [KRaft Mode Guide](https://kafka.apache.org/documentation/#kraft)
