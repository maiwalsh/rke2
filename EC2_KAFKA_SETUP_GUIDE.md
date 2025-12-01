# RKE2 on EC2 with Confluent Kafka Deployment Guide

This guide walks you through setting up an RKE2 Kubernetes cluster on AWS EC2 instances and deploying Confluent Kafka.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Prepare EC2 Instances](#step-1-prepare-ec2-instances)
- [Step 2: Install RKE2 Server (Control Plane)](#step-2-install-rke2-server-control-plane)
- [Step 3: Install RKE2 Agent (Worker Nodes)](#step-3-install-rke2-agent-worker-nodes)
- [Step 4: Configure kubectl Access](#step-4-configure-kubectl-access)
- [Step 5: Deploy Confluent Kafka](#step-5-deploy-confluent-kafka)
- [Step 6: Verify and Test Kafka](#step-6-verify-and-test-kafka)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### AWS EC2 Requirements

**Minimum Setup:**
- 1 server node (control plane): t3.medium or larger (2 vCPU, 4GB RAM)
- 2+ worker nodes: t3.medium or larger (2 vCPU, 4GB RAM)

**Operating System:**
- Ubuntu 20.04/22.04 LTS, RHEL 8/9, or Amazon Linux 2023

**Security Group Configuration:**
Open the following ports:

**Server Node:**
- 22/tcp - SSH
- 6443/tcp - Kubernetes API
- 9345/tcp - RKE2 supervisor API
- 10250/tcp - Kubelet metrics
- 2379-2380/tcp - etcd client and peer
- 8472/udp - Flannel VXLAN (if using Flannel)
- 4240/tcp - Cilium health checks (if using Cilium)

**Worker Nodes:**
- 22/tcp - SSH
- 10250/tcp - Kubelet metrics
- 8472/udp - Flannel VXLAN
- 30000-32767/tcp - NodePort Services

**All Nodes:**
- Allow all traffic between cluster nodes (private subnet recommended)

---

## Step 1: Prepare EC2 Instances

### 1.1 Launch EC2 Instances

```bash
# Example using AWS CLI (adjust as needed)
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --count 3 \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=rke2-node}]'
```

### 1.2 SSH into All Nodes

```bash
ssh -i your-key.pem ubuntu@<server-node-ip>
```

### 1.3 Update System (on all nodes)

```bash
# For Ubuntu/Debian
sudo apt-get update && sudo apt-get upgrade -y

# For RHEL/CentOS/Amazon Linux
sudo yum update -y
```

### 1.4 Disable Swap (required for Kubernetes)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 1.5 Set Hostnames (optional but recommended)

```bash
# On server node
sudo hostnamectl set-hostname rke2-server

# On worker nodes
sudo hostnamectl set-hostname rke2-worker-1
sudo hostnamectl set-hostname rke2-worker-2
```

---

## Step 2: Install RKE2 Server (Control Plane)

Run these commands on your designated **server node**.

### 2.1 Download and Install RKE2

```bash
# Download the RKE2 installation script
curl -sfL https://get.rke2.io | sudo sh -
```

### 2.2 Create RKE2 Configuration

```bash
# Create config directory
sudo mkdir -p /etc/rancher/rke2

# Create configuration file
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
write-kubeconfig-mode: "0644"
tls-san:
  - "$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)"
  - "$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)"
node-label:
  - "node-role.kubernetes.io/server=true"
EOF
```

### 2.3 Enable and Start RKE2 Server

```bash
# Enable the service to start on boot
sudo systemctl enable rke2-server.service

# Start the service now
sudo systemctl start rke2-server.service

# Check status
sudo systemctl status rke2-server.service
```

**On RHEL/CentOS/Amazon Linux, the service will likely be failing.** Check the logs:

```bash
sudo journalctl -u rke2-server -f
```

**Expected issue:** Errors about `nm-cloud-setup.service` with the service repeatedly restarting.

Press `Ctrl+C` to stop watching logs.

#### Understanding the Networking Conflict

**Why does this fail on RHEL-based systems?**

RKE2 needs complete control over networking to configure Kubernetes CNI (Container Network Interface). This includes:
- Creating virtual network interfaces (veth pairs)
- Setting up routing tables and iptables rules
- Managing IP address allocation for pods
- Configuring network policies

On RHEL/CentOS/Amazon Linux, the `nm-cloud-setup` service (part of NetworkManager) automatically configures networking for cloud instances. If both services try to manage networking simultaneously, they create conflicting configurations that break cluster communication.

RKE2 proactively checks for this conflict and refuses to start if `nm-cloud-setup` is enabled, preventing a broken cluster.

#### Resolving the Conflict

Disable nm-cloud-setup so RKE2 can manage networking:

```bash
# Stop RKE2 (already failing, but be explicit)
sudo systemctl stop rke2-server.service

# Disable and stop NetworkManager cloud setup
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo systemctl stop nm-cloud-setup.service nm-cloud-setup.timer

# Verify it's disabled
sudo systemctl status nm-cloud-setup.service
# Should show: "disabled" and "inactive (dead)"

# Start RKE2 again
sudo systemctl start rke2-server.service

# Monitor startup - should succeed this time
sudo journalctl -u rke2-server -f
```

**Successful startup indicators:**
- "Starting containerd"
- "Starting etcd"
- "Node registered successfully"
- Logs stabilize after 1-2 minutes

Press `Ctrl+C` when stable.

### 2.4 Configure kubectl Access

**Why we can't use kubectl directly:**

RKE2 runs as root and creates `/etc/rancher/rke2/rke2.yaml` with root-only permissions. Your user account can't read this file. While you could use `sudo kubectl` for every command, this is inconvenient and non-standard.

**Standard Kubernetes practice:** Copy the kubeconfig to `~/.kube/config` in your home directory where all Kubernetes tools expect to find it.

```bash
# Create standard .kube directory
mkdir -p ~/.kube

# Copy kubeconfig from RKE2's location
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

# Change ownership to your user
# $(id -u) = your user ID, $(id -g) = your group ID
sudo chown $(id -u):$(id -g) ~/.kube/config

# Secure the file (Kubernetes security best practice)
chmod 600 ~/.kube/config
```

**Configure environment:**

```bash
# Make kubectl and config available in all sessions
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc

# Apply to current session
source ~/.bashrc

# Verify cluster access
kubectl get nodes
```

**Expected output:**
```
NAME                    STATUS   ROLES                       AGE   VERSION
your-node-name          Ready    control-plane,etcd,master   2m    v1.33.x+rke2
```

**If node shows "NotReady":** Wait 30-60 seconds for CNI initialization, then check again.

### 2.5 Get the Node Token

You'll need this token to join worker nodes:

```bash
sudo cat /var/lib/rancher/rke2/server/node-token
```

**Save this token** - you'll need it for worker nodes!

---

## Step 3: Install RKE2 Agent (Worker Nodes)

Run these commands on each **worker node**.

### 3.1 Download and Install RKE2 Agent

```bash
# Set the installation type to agent
export INSTALL_RKE2_TYPE="agent"

# Download and install
curl -sfL https://get.rke2.io | sudo sh -
```

### 3.2 Configure RKE2 Agent

```bash
# Create config directory
sudo mkdir -p /etc/rancher/rke2

# Create configuration file
# Replace <SERVER_IP> with your server node's private IP
# Replace <NODE_TOKEN> with the token from step 2.5
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
server: https://<SERVER_IP>:9345
token: <NODE_TOKEN>
node-label:
  - "node-role.kubernetes.io/worker=true"
EOF
```

### 3.3 Enable and Start RKE2 Agent

```bash
# Enable the service
sudo systemctl enable rke2-agent.service

# Start the service
sudo systemctl start rke2-agent.service

# Check status
sudo systemctl status rke2-agent.service
```

### 3.4 Verify Worker Nodes

From the **server node**, check that all nodes have joined:

```bash
kubectl get nodes
```

You should see all nodes in a "Ready" state.

---

## Step 4: Configure kubectl Access

### 4.1 Set Up kubectl on Server Node

```bash
# Add to your ~/.bashrc or ~/.bash_profile
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
source ~/.bashrc
```

### 4.2 (Optional) Configure kubectl from Your Local Machine

```bash
# On the server node, copy the kubeconfig
sudo cat /etc/rancher/rke2/rke2.yaml

# On your local machine, save to ~/.kube/config
# Replace 127.0.0.1 with your server's public IP
# Then test:
kubectl get nodes
```

---

## Step 5: Deploy Confluent Kafka

### 5.1 Install Helm (if not already installed)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 5.2 Add Confluent Helm Repository

```bash
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
```

### 5.3 Create a Namespace for Kafka

```bash
kubectl create namespace kafka
```

### 5.4 Create Kafka Configuration File

Create a `kafka-values.yaml` file with custom settings:

```bash
cat > kafka-values.yaml <<EOF
cp-zookeeper:
  enabled: true
  servers: 3
  persistence:
    enabled: true
    dataDirStorageClass: "local-path"
    dataDirSize: 5Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi

cp-kafka:
  enabled: true
  brokers: 3
  persistence:
    enabled: true
    storageClass: "local-path"
    size: 5Gi
  resources:
    requests:
      cpu: 250m
      memory: 1Gi
  configurationOverrides:
    "offsets.topic.replication.factor": "3"
    "default.replication.factor": "3"
    "min.insync.replicas": "2"

cp-schema-registry:
  enabled: true
  replicaCount: 1

cp-kafka-rest:
  enabled: true

cp-kafka-connect:
  enabled: false

cp-ksql-server:
  enabled: false

cp-control-center:
  enabled: false
EOF
```

### 5.5 Install Kafka using Helm

```bash
helm install kafka confluentinc/cp-helm-charts \
  --namespace kafka \
  --values kafka-values.yaml
```

### 5.6 Wait for Pods to be Ready

```bash
# Watch the pods come up
kubectl get pods -n kafka -w

# It may take 3-5 minutes for all pods to be ready
```

---

## Step 6: Verify and Test Kafka

### 6.1 Check Pod Status

```bash
kubectl get pods -n kafka
```

All pods should show `Running` status.

### 6.2 Get Kafka Services

```bash
kubectl get svc -n kafka
```

### 6.3 Test Kafka - Create a Topic

```bash
# Get the Kafka broker pod name
KAFKA_POD=$(kubectl get pod -n kafka -l app=cp-kafka -o jsonpath='{.items[0].metadata.name}')

# Create a test topic
kubectl exec -n kafka $KAFKA_POD -- kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 3

# List topics
kubectl exec -n kafka $KAFKA_POD -- kafka-topics \
  --bootstrap-server localhost:9092 \
  --list
```

### 6.4 Test Producer and Consumer

**Producer:**
```bash
# Produce messages
kubectl exec -n kafka $KAFKA_POD -- bash -c "echo 'Hello Kafka from RKE2!' | kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test-topic"
```

**Consumer:**
```bash
# Consume messages
kubectl exec -n kafka $KAFKA_POD -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 1
```

### 6.5 Expose Kafka (Optional)

To access Kafka from outside the cluster:

```bash
# Create a NodePort service
kubectl expose service kafka-cp-kafka \
  --name=kafka-external \
  --namespace=kafka \
  --type=NodePort \
  --port=9092 \
  --target-port=9092

# Get the NodePort
kubectl get svc kafka-external -n kafka
```

Then connect using: `<any-node-ip>:<nodeport>`

---

## Troubleshooting

### Node Not Ready

```bash
# Check node status
kubectl describe node <node-name>

# Check RKE2 logs
sudo journalctl -u rke2-server -f  # on server
sudo journalctl -u rke2-agent -f   # on worker
```

### Pods Stuck in Pending

```bash
# Check pod events
kubectl describe pod <pod-name> -n kafka

# Check storage
kubectl get pv
kubectl get pvc -n kafka
```

### Kafka Connection Issues

```bash
# Check Zookeeper first
kubectl logs -n kafka <zookeeper-pod-name>

# Check Kafka broker logs
kubectl logs -n kafka <kafka-pod-name>

# Test internal connectivity
kubectl run -it --rm kafka-test --image=confluentinc/cp-kafka:latest --restart=Never -- bash
# Inside the container:
kafka-broker-api-versions --bootstrap-server kafka-cp-kafka.kafka.svc.cluster.local:9092
```

### Insufficient Resources

```bash
# Check resource usage
kubectl top nodes
kubectl top pods -n kafka

# If nodes are running out of resources, either:
# 1. Reduce Kafka resource requests in kafka-values.yaml
# 2. Add more worker nodes
# 3. Use larger EC2 instance types
```

### Security Group Issues

Ensure all required ports are open between nodes. Test connectivity:

```bash
# From worker to server (port 6443 and 9345)
nc -zv <server-ip> 6443
nc -zv <server-ip> 9345
```

---

## Clean Up (Optional)

To remove everything:

```bash
# Delete Kafka
helm uninstall kafka -n kafka
kubectl delete namespace kafka

# Stop RKE2 on all nodes
sudo systemctl stop rke2-server  # on server
sudo systemctl stop rke2-agent   # on workers

# Uninstall RKE2 (if needed)
sudo /usr/local/bin/rke2-uninstall.sh  # on server
sudo /usr/local/bin/rke2-agent-uninstall.sh  # on workers
```

---

## Next Steps

- Set up monitoring with Prometheus and Grafana
- Configure ingress for external access
- Implement backup strategies for Kafka data
- Set up Kafka Connect for data integration
- Enable SSL/TLS for Kafka brokers
- Configure RBAC and pod security policies

---

## Additional Resources

- [RKE2 Official Documentation](https://docs.rke2.io/)
- [Confluent Helm Charts](https://github.com/confluentinc/cp-helm-charts)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kafka Documentation](https://kafka.apache.org/documentation/)

---

**Happy Clustering! 🚀**
