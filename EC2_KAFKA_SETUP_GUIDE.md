# RKE2 on EC2 with Confluent Kafka - Complete Deployment Guide

This guide provides step-by-step instructions for deploying RKE2 Kubernetes cluster on AWS EC2 and running Confluent Kafka with KRaft mode (ZooKeeper-free).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Part 1: EC2 Instance Setup](#part-1-ec2-instance-setup)
- [Part 2: Install RKE2](#part-2-install-rke2)
- [Part 3: Configure kubectl Access](#part-3-configure-kubectl-access)
- [Part 4: Deploy Confluent Kafka with KRaft](#part-4-deploy-confluent-kafka-with-kraft)
- [Part 5: Verify and Test](#part-5-verify-and-test)
- [Troubleshooting](#troubleshooting)
- [Reference](#reference)

---

## Prerequisites

### AWS EC2 Requirements

**Instance Type:**
- **Minimum:** t3.medium (2 vCPU, 4GB RAM)
- **Recommended:** t3.2xlarge (8 vCPU, 32GB RAM)

**Disk Size (CRITICAL):**
- **Minimum:** 30GB EBS root volume
- **Recommended:** 100GB EBS root volume

**Why disk size matters:**
```
RHEL 8 base OS:              ~4GB
RKE2 + containerd:           ~3GB
Kafka container images:      ~2-3GB per image × 6 pods = ~18GB
Kafka persistent data:       3 controllers × 10GB + 3 brokers × 10GB = 60GB
System overhead:             ~2GB
────────────────────────────────
Total:                       ~87GB
```

**Operating System:**
- RHEL 8.x, CentOS 8.x, Amazon Linux 2023, or Ubuntu 20.04/22.04 LTS

**Security Group Configuration:**

| Port | Protocol | Purpose | Source |
|------|----------|---------|--------|
| 22 | TCP | SSH | Your IP |
| 6443 | TCP | Kubernetes API | Your IP / VPC CIDR |
| 9345 | TCP | RKE2 supervisor API (for worker nodes) | VPC CIDR |
| 10250 | TCP | Kubelet metrics | VPC CIDR |
| 30000-32767 | TCP | NodePort Services (if exposing Kafka externally) | Your IP |

---

## Part 1: EC2 Instance Setup

### 1.1 Launch EC2 Instance

```bash
# Example using AWS CLI
aws ec2 run-instances \
  --image-id ami-xxxxxx \
  --count 1 \
  --instance-type t3.2xlarge \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":100,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=rke2-kafka-node}]'
```

**Critical:** Set `VolumeSize` to at least 100 for the root volume.

### 1.2 Connect to Instance

```bash
# For RHEL/Amazon Linux
ssh -i your-key.pem ec2-user@<public-ip>

# For Ubuntu
ssh -i your-key.pem ubuntu@<public-ip>
```

### 1.3 Update System

```bash
# RHEL/CentOS/Amazon Linux
sudo yum update -y

# Ubuntu
sudo apt-get update && sudo apt-get upgrade -y
```

### 1.4 Verify Disk Space

```bash
df -h /
```

**Expected output:** Size should show at least 30GB, ideally 100GB.

---

## Part 2: Install RKE2

### 2.1 Download and Install RKE2

```bash
curl -sfL https://get.rke2.io | sudo sh -
```

**What this installs:**
- RKE2 binaries in `/var/lib/rancher/rke2/bin/`
- Systemd service files
- Containerd runtime
- kubectl, crictl utilities

### 2.2 Enable RKE2 Service

```bash
sudo systemctl enable rke2-server.service
```

### 2.3 Handle NetworkManager Conflict (RHEL/CentOS/Amazon Linux Only)

**Skip this section if using Ubuntu.**

#### 2.3.1 Why This is Necessary

RKE2 requires exclusive control over network configuration to manage:
- Container Network Interface (CNI) for pod networking
- Virtual network interfaces (veth pairs)
- IP routing tables and iptables rules
- Network policies

RHEL-based systems include `nm-cloud-setup` service that auto-configures networking for cloud instances. This conflicts with RKE2's CNI, causing the service to fail repeatedly.

#### 2.3.2 Disable nm-cloud-setup

```bash
# Disable and stop NetworkManager cloud setup
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo systemctl stop nm-cloud-setup.service nm-cloud-setup.timer

# Verify it's disabled
sudo systemctl status nm-cloud-setup.service
```

**Expected output:** Should show "disabled" and "inactive (dead)".

### 2.4 Start RKE2

```bash
sudo systemctl start rke2-server.service
```

### 2.5 Verify RKE2 is Running

```bash
# Check service status
sudo systemctl status rke2-server.service

# Watch logs (Ctrl+C to exit)
sudo journalctl -u rke2-server -f
```

**Success indicators:**
```
... Starting containerd ...
... Starting etcd ...
... Node registered successfully ...
```

**Startup time:** 1-2 minutes for first start (longer on slower instances).

---

## Part 3: Configure kubectl Access

### 3.1 Understanding the Permission Issue

RKE2 runs as root and creates `/etc/rancher/rke2/rke2.yaml` with root-only permissions (600). Your user account cannot read this file.

**Why not just chmod?**
- RKE2 owns and may update this file
- Standard Kubernetes practice is `~/.kube/config` in user home directory
- All Kubernetes tools default to looking there

### 3.2 Copy kubeconfig to User Home

```bash
# Create .kube directory
mkdir -p ~/.kube

# Copy kubeconfig
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

# Change ownership to your user
sudo chown $(id -u):$(id -g) ~/.kube/config

# Secure the file (best practice)
chmod 600 ~/.kube/config
```

### 3.3 Configure Environment

```bash
# Add to .bash_profile (for persistent sessions)
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bash_profile
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bash_profile

# Apply to current session
source ~/.bash_profile
```

**Why .bash_profile instead of .bashrc?**
- `.bash_profile` loads first on login shells
- `.bashrc` may not run or may be overwritten by `.bash_profile`
- Using `.bash_profile` ensures kubectl works after re-login

### 3.4 Verify kubectl Works

```bash
kubectl get nodes
```

**Expected output:**
```
NAME                  STATUS   ROLES                       AGE   VERSION
ip-172-31-xx-xx       Ready    control-plane,etcd,master   2m    v1.33.x+rke2
```

---

## Part 4: Deploy Confluent Kafka with KRaft

### 4.1 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

### 4.2 Configure Storage Class

RKE2 includes the `local-path` storage provisioner, but it's not set as default.

```bash
# Set local-path as default storage class
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
```

**Expected output:**
```
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  5m
```

**What this does:**
- Enables automatic persistent volume creation
- Without default storage class, Kafka pods stay "Pending"
- Local-path creates volumes on node disk at `/opt/local-path-provisioner/`

### 4.3 Download Confluent for Kubernetes

Confluent for Kubernetes (CFK) is **not** available in public Helm repositories. It must be downloaded as a bundle.

```bash
# Download CFK bundle (~218MB)
curl -O https://packages.confluent.io/bundle/cfk/confluent-for-kubernetes-3.1.0.tar.gz

# Extract
tar -xvf confluent-for-kubernetes-3.1.0.tar.gz

# Verify extraction
ls -la confluent-for-kubernetes-3.1.0-*/
```

### 4.4 Install Confluent for Kubernetes Operator

```bash
# Install from extracted bundle
helm upgrade --install confluent-operator \
  ./confluent-for-kubernetes-3.1.0-*/helm/confluent-for-kubernetes \
  --set kRaftEnabled=true \
  --namespace confluent \
  --create-namespace
```

**Parameters explained:**
- `kRaftEnabled=true`: Enables KRaft controller support in operator
- `--namespace confluent`: Creates dedicated namespace for Confluent components
- `--create-namespace`: Creates namespace if it doesn't exist

**Verify operator is running:**
```bash
kubectl get pods -n confluent
```

**Expected output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
confluent-operator-xxxxx-xxxxx       1/1     Running   0          30s
```

### 4.5 Deploy KRaft Controllers and Kafka Brokers

Create the Kafka cluster configuration:

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

**Configuration breakdown:**

**KRaftController:**
- `replicas: 3` - Creates 3 controller nodes (odd number required for Raft consensus)
- `application: confluentinc/cp-server:7.9.1` - Official Confluent Platform image
- `init: confluentinc/confluent-init-container:2.9.1` - Init container for configuration
- `dataVolumeCapacity: 10Gi` - 10GB persistent storage per controller
- Controllers handle cluster metadata, leader elections, and coordination

**Kafka:**
- `replicas: 3` - Creates 3 broker nodes
- Brokers handle message storage, replication, and client connections
- `dependencies.kRaftController` - Links brokers to controller cluster

### 4.6 Apply Configuration

```bash
kubectl apply -f confluent-kafka-kraft.yaml
```

### 4.7 Watch Deployment

```bash
kubectl get pods -n confluent -w
```

**Deployment sequence:**
1. **kraftcontroller-0, -1, -2**: Init:0/1 → PodInitializing → Running (2-3 minutes each)
2. **kafka-0, -1, -2**: Init:0/1 → PodInitializing → Running (2-3 minutes each, after controllers are ready)

**Total deployment time:** 5-8 minutes depending on network speed.

Press `Ctrl+C` once all pods show "1/1 Running".

### 4.8 Verify All Components

```bash
kubectl get all -n confluent
```

**Expected resources:**
- 1 operator pod
- 3 KRaft controller pods
- 3 Kafka broker pods
- Services for kafka, kraftcontroller
- 6 PersistentVolumeClaims (10Gi each)

---

## Part 5: Verify and Test

### 5.1 Create Test Client Pod

```bash
kubectl run kafka-client -n confluent \
  --image=confluentinc/cp-server:7.9.1 \
  --restart=Never \
  --command -- sleep infinity

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/kafka-client -n confluent --timeout=60s
```

### 5.2 Create Test Topic

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 3
```

**Expected output:**
```
Created topic test-topic.
```

### 5.3 Describe Topic

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --describe \
  --topic test-topic
```

**Expected output:**
```
Topic: test-topic       TopicId: xxxxx  PartitionCount: 3       ReplicationFactor: 3
        Topic: test-topic       Partition: 0    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
        Topic: test-topic       Partition: 1    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
        Topic: test-topic       Partition: 2    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
```

### 5.4 Produce Messages

```bash
kubectl exec -n confluent kafka-client -- bash -c \
  "echo 'Message 1: Hello from Confluent Kafka on RKE2' | kafka-console-producer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic"

kubectl exec -n confluent kafka-client -- bash -c \
  "echo 'Message 2: KRaft mode - no ZooKeeper needed!' | kafka-console-producer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic"
```

### 5.5 Consume Messages

```bash
kubectl exec -n confluent kafka-client -- kafka-console-consumer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 2
```

**Expected output:**
```
Message 1: Hello from Confluent Kafka on RKE2
Message 2: KRaft mode - no ZooKeeper needed!
Processed a total of 2 messages
```

**If you see both messages, Kafka is working correctly!** ✅

---

## Troubleshooting

### Issue 1: Pods Stuck in "Pending" - Disk Pressure

**Symptoms:**
```bash
$ kubectl get pods -n confluent
NAME                STATUS    RESTARTS   AGE
kafka-0             Pending   0          5m

$ kubectl describe node | grep Taints
Taints:  node.kubernetes.io/disk-pressure:NoSchedule
```

**Root cause:** Disk usage exceeded 85% threshold. Kubernetes applies taint to prevent scheduling new pods.

**Diagnosis:**
```bash
# Check disk usage
df -h /

# Check for leftover PVCs from failed deployments
kubectl get pvc --all-namespaces

# Check PersistentVolumes
kubectl get pv
```

**Solution 1: Expand EBS Volume**
```bash
# In AWS Console:
# EC2 → Volumes → Select volume → Actions → Modify Volume → Increase size to 100GB

# On EC2 instance:
sudo growpart /dev/nvme0n1 3
sudo xfs_growfs /

# Verify
df -h /
# Should show new size
```

**Solution 2: Clean Up Leftover PVCs**
```bash
# List all PVCs
kubectl get pvc --all-namespaces

# Delete PVCs from old deployments
kubectl delete pvc <pvc-name> -n <namespace>

# Check for Released PVs and delete them
kubectl get pv
kubectl delete pv <pv-name>
```

**Solution 3: Prune Container Images**
```bash
sudo /var/lib/rancher/rke2/bin/crictl \
  --runtime-endpoint unix:///run/k3s/containerd/containerd.sock \
  rmi --prune
```

**Solution 4: Remove Taint Manually (after fixing disk)**
```bash
kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule-
```

**Solution 5: Delete and Recreate Pending Pods**
```bash
# Delete pods stuck in Pending
kubectl delete pod <pod-name> -n confluent

# Deployment will recreate them automatically
```

### Issue 2: kubectl Not Found After Re-login

**Symptom:** After logout/login, `kubectl: command not found`

**Root cause:** `.bash_profile` runs before `.bashrc` and may overwrite PATH.

**Solution:**
```bash
# Add to .bash_profile instead of .bashrc
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bash_profile
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bash_profile
source ~/.bash_profile
```

### Issue 3: No Storage Class Available

**Symptom:** Pods stuck in Pending with message "unbound immediate PersistentVolumeClaims"

**Solution:**
```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Issue 4: RKE2 Won't Start on RHEL/CentOS

**Symptom:**
```bash
$ sudo journalctl -u rke2-server -f
... error about nm-cloud-setup.service ...
... service restart counter at 100+ ...
```

**Solution:** Disable nm-cloud-setup (see Part 2, Section 2.3)

### Issue 5: Pods in "Evicted" Status

**Symptom:** `kubectl get pods` shows multiple pods with "Evicted" status

**Root cause:** Kubernetes evicted pods due to resource pressure (disk/memory).

**Solution:**
```bash
# Delete evicted pods (they're already dead)
kubectl get pods -n confluent | grep Evicted | awk '{print $1}' | \
  xargs kubectl delete pod -n confluent

# Fix underlying cause (usually disk pressure)
```

### Issue 6: "ContainerStatusUnknown"

**Symptom:** Pods stuck in "ContainerStatusUnknown" status

**Root cause:** Node was under heavy resource pressure, kubelet lost connection to pods.

**Solution:**
```bash
# Delete affected pods
kubectl delete pods -n confluent --all

# Deployment will recreate them
# Fix underlying resource issue (usually disk pressure)
```

---

## Reference

### Service Endpoints

**Internal Kafka Access (from pods):**
```
kafka.confluent.svc.cluster.local:9092
```

**KRaft Controller (internal only):**
```
kraftcontroller.confluent.svc.cluster.local:9093
```

### Key Commands

**Check cluster health:**
```bash
kubectl get nodes
kubectl get pods -n confluent
kubectl get pvc -n confluent
kubectl get storageclass
```

**View logs:**
```bash
# Kafka broker logs
kubectl logs -n confluent kafka-0

# Controller logs
kubectl logs -n confluent kraftcontroller-0

# Operator logs
kubectl logs -n confluent deployment/confluent-operator -f
```

**Scale cluster:**
```bash
# Edit resource to change replicas
kubectl edit kafka kafka -n confluent

# Or update YAML and reapply
kubectl apply -f confluent-kafka-kraft.yaml
```

**Delete everything:**
```bash
kubectl delete -f confluent-kafka-kraft.yaml
helm uninstall confluent-operator -n confluent
kubectl delete namespace confluent
```

### File Locations

**RKE2:**
- Config: `/etc/rancher/rke2/config.yaml`
- Kubeconfig: `/etc/rancher/rke2/rke2.yaml`
- Data directory: `/var/lib/rancher/rke2/`
- Binaries: `/var/lib/rancher/rke2/bin/`
- Logs: `journalctl -u rke2-server`

**Kafka Storage:**
- PVC mount point: `/opt/local-path-provisioner/pvc-<uuid>_<namespace>_<pvc-name>/`

### Images Used

- **Confluent Kafka:** `confluentinc/cp-server:7.9.1`
- **Init container:** `confluentinc/confluent-init-container:2.9.1`
- **Operator:** `docker.io/confluentinc/confluent-operator:0.949.1` (automatically pulled by Helm)

### Resource Requirements

**Per Pod:**
- KRaft Controller: ~512MB RAM, 0.5 CPU, 10Gi disk
- Kafka Broker: ~1GB RAM, 1 CPU, 10Gi disk
- Operator: ~256MB RAM, 0.1 CPU

**Total for 3-node cluster:**
- ~7GB RAM
- ~4.5 CPU
- 60Gi disk (persistent volumes)

### API Versions

- Confluent for Kubernetes: `platform.confluent.io/v1beta1`
- Storage Class: `storage.k8s.io/v1`
- Kubernetes: `v1.33.x+rke2`

---

## Additional Resources

- [Confluent for Kubernetes Documentation](https://docs.confluent.io/operator/current/overview.html)
- [RKE2 Documentation](https://docs.rke2.io/)
- [Kafka KRaft Mode](https://kafka.apache.org/documentation/#kraft)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

---

**End of Guide**
