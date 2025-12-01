# Confluent Kafka on RKE2 - Deployment Runbook

**Purpose:** Deploy Confluent Kafka with KRaft mode on an existing RKE2 cluster

**Time to Complete:** 10-15 minutes (excluding image pull time)

---

## Prerequisites

- RKE2 cluster running and healthy
- kubectl configured and working
- Helm 3 installed
- 100GB+ disk space available
- t3.2xlarge instance (or equivalent: 8 vCPU, 32GB RAM)

**Verify prerequisites:**
```bash
kubectl get nodes
# Should show: Ready status
```

---

## Step 1: Configure Storage Class

Set local-path as the default storage class:

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Verify:**
```bash
kubectl get storageclass
```
Expected output: `local-path (default)`

---

## Step 2: Download Confluent for Kubernetes

```bash
# Download CFK bundle (218MB)
curl -O https://packages.confluent.io/bundle/cfk/confluent-for-kubernetes-3.1.0.tar.gz

# Extract
tar -xvf confluent-for-kubernetes-3.1.0.tar.gz
```

**Verify extraction:**
```bash
ls -la confluent-for-kubernetes-3.1.0-*/
```

---

## Step 3: Install Confluent for Kubernetes Operator

```bash
helm upgrade --install confluent-operator \
  ./confluent-for-kubernetes-3.1.0-*/helm/confluent-for-kubernetes \
  --set kRaftEnabled=true \
  --namespace confluent \
  --create-namespace
```

**Verify operator is running:**
```bash
kubectl get pods -n confluent
```
Wait until: `confluent-operator-xxxxx` shows `1/1 Running`

---

## Step 4: Deploy Kafka Cluster with KRaft

Create Kafka configuration:

```bash
cat > confluent-kafka-kraft.yaml << 'EOF'
---
apiVersion: platform.confluent.io/v1beta1
kind: KRaftController
metadata:
  name: kraftcontroller
  namespace: confluent
spec:
  replicas: 3
  image:
    application: confluentinc/cp-server:7.9.1
    init: confluentinc/confluent-init-container:2.9.1
  dataVolumeCapacity: 10Gi
  storageClass:
    name: local-path
---
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent
spec:
  replicas: 3
  image:
    application: confluentinc/cp-server:7.9.1
    init: confluentinc/confluent-init-container:2.9.1
  dataVolumeCapacity: 10Gi
  storageClass:
    name: local-path
  dependencies:
    kRaftController:
      clusterRef:
        name: kraftcontroller
EOF
```

**Apply configuration:**
```bash
kubectl apply -f confluent-kafka-kraft.yaml
```

---

## Step 5: Monitor Deployment

Watch pods come up:

```bash
kubectl get pods -n confluent -w
```

**Expected sequence:**
1. `kraftcontroller-0, -1, -2`: Init → PodInitializing → Running (2-3 min each)
2. `kafka-0, -1, -2`: Init → PodInitializing → Running (2-3 min each)

**Total time:** 5-8 minutes

Press `Ctrl+C` when all pods show `1/1 Running`

---

## Step 6: Verify Deployment

Check all components:

```bash
kubectl get pods -n confluent
```

**Expected output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
confluent-operator-xxxxx             1/1     Running   0          10m
kafka-0                              1/1     Running   0          5m
kafka-1                              1/1     Running   0          5m
kafka-2                              1/1     Running   0          5m
kraftcontroller-0                    1/1     Running   0          7m
kraftcontroller-1                    1/1     Running   0          7m
kraftcontroller-2                    1/1     Running   0          7m
```

**Check PVCs:**
```bash
kubectl get pvc -n confluent
```
Should show 6 PVCs (3 for controllers, 3 for brokers), all `Bound`

---

## Step 7: Test Kafka Cluster

### Create test client pod:

```bash
kubectl run kafka-client -n confluent \
  --image=confluentinc/cp-server:7.9.1 \
  --restart=Never \
  --command -- sleep infinity

kubectl wait --for=condition=ready pod/kafka-client -n confluent --timeout=60s
```

### Create test topic:

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 3
```

Expected: `Created topic test-topic.`

### Verify topic:

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --describe \
  --topic test-topic
```

Should show 3 partitions, each with 3 replicas.

### Produce message:

```bash
kubectl exec -n confluent kafka-client -- bash -c \
  'echo "Test message from Confluent Kafka" | kafka-console-producer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic'
```

### Consume message:

```bash
kubectl exec -n confluent kafka-client -- kafka-console-consumer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 1
```

Expected output: `Test message from Confluent Kafka`

**✅ If you see the message, Kafka is working correctly!**

---

## Kafka Connection Details

**Internal (from pods in cluster):**
```
bootstrap.servers=kafka.confluent.svc.cluster.local:9092
```

**External (for development/testing only):**

Option 1 - NodePort:
```bash
kubectl expose service kafka \
  --name=kafka-external \
  --namespace=confluent \
  --type=NodePort \
  --port=9092

kubectl get svc kafka-external -n confluent
```
Connect using: `<ec2-public-ip>:<nodeport>`

Option 2 - Port Forward:
```bash
kubectl port-forward -n confluent svc/kafka 9092:9092
```
Connect using: `localhost:9092`

---

## Common Operations

### View cluster status:
```bash
kubectl get pods -n confluent
kubectl get pvc -n confluent
kubectl get svc -n confluent
```

### View logs:
```bash
# Kafka broker
kubectl logs -n confluent kafka-0

# Controller
kubectl logs -n confluent kraftcontroller-0

# Operator
kubectl logs -n confluent deployment/confluent-operator -f
```

### Scale Kafka brokers:
```bash
kubectl edit kafka kafka -n confluent
# Change spec.replicas: 3 to desired number (odd numbers recommended)
```

### Delete test client:
```bash
kubectl delete pod kafka-client -n confluent
```

---

## Troubleshooting

### Issue: Pods stuck in "Pending"

**Check disk space:**
```bash
df -h /
```
If >80% full, expand EBS volume in AWS Console, then:
```bash
sudo growpart /dev/nvme0n1 3
sudo xfs_growfs /
```

**Check for disk pressure taint:**
```bash
kubectl describe node | grep Taints
```
If disk pressure taint exists:
```bash
kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule-
```

**Check PVCs:**
```bash
kubectl get pvc -n confluent
```
All should be `Bound`. If stuck, check storage class:
```bash
kubectl get storageclass
```

### Issue: Pods stuck in "ContainerCreating"

**This is normal.** Each pod downloads ~2GB images. Check progress:
```bash
kubectl describe pod <pod-name> -n confluent
```
Look for "pulling image" messages. Wait 3-5 minutes.

### Issue: Kafka not responding

**Check if all pods are running:**
```bash
kubectl get pods -n confluent
```

**Check service endpoints:**
```bash
kubectl get endpoints kafka -n confluent
```
Should show 3 IPs (one for each broker).

**Check broker logs:**
```bash
kubectl logs -n confluent kafka-0 --tail=50
```

### Issue: Can't produce/consume messages

**Test connectivity from client pod:**
```bash
kubectl exec -n confluent kafka-client -- kafka-broker-api-versions \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092
```
Should list available brokers.

**Check topic exists:**
```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --list
```

---

## Cleanup (if needed)

**To remove Kafka cluster:**
```bash
kubectl delete -f confluent-kafka-kraft.yaml
```

**To remove operator:**
```bash
helm uninstall confluent-operator -n confluent
```

**To remove everything including namespace:**
```bash
kubectl delete namespace confluent
```

---

## Architecture

**Components:**
- **3 KRaft Controllers:** Manage cluster metadata (replaces ZooKeeper)
- **3 Kafka Brokers:** Handle message storage and client connections
- **1 Operator:** Manages Kafka resources and lifecycle

**Storage:**
- 10GB persistent volume per controller (30GB total)
- 10GB persistent volume per broker (30GB total)
- Total: 60GB storage allocated

**Resources per pod:**
- Controller: ~512MB RAM, 0.5 CPU
- Broker: ~1GB RAM, 1 CPU
- Total cluster: ~7GB RAM, ~4.5 CPU

---

## Important Notes

1. **KRaft Mode:** No ZooKeeper required (deprecated November 2025)
2. **Official Images:** Uses Confluent's official cp-server:7.9.1
3. **High Availability:** 3 replicas provide fault tolerance
4. **Persistent Storage:** Data survives pod restarts
5. **Disk Space:** Monitor disk usage - Kubernetes taints nodes at 85%

---

## Support Information

**Official Documentation:**
- Confluent for Kubernetes: https://docs.confluent.io/operator/current/overview.html
- Kafka KRaft: https://kafka.apache.org/documentation/#kraft

**Key Files:**
- Configuration: `confluent-kafka-kraft.yaml`
- Operator chart: `./confluent-for-kubernetes-3.1.0-*/helm/confluent-for-kubernetes`

**Images:**
- Kafka/Controller: `confluentinc/cp-server:7.9.1`
- Init Container: `confluentinc/confluent-init-container:2.9.1`

---

**End of Runbook**
