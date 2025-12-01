# Simple Walkthrough: RKE2 on EC2 with Confluent Kafka

This is a straightforward guide that explains **what** you're doing and **why** at each step.

---

## Understanding the Goal

You want to:
1. Install RKE2 (a Kubernetes distribution) on your EC2 instance
2. Use that Kubernetes cluster to run Confluent Kafka with KRaft mode (no ZooKeeper)
3. Understand how everything connects

---

## Part 0: EC2 Instance Setup (IMPORTANT)

### Disk Size Requirements

**Critical:** Your EC2 instance needs adequate disk space. Here's why:

- **RHEL/CentOS base OS:** ~4GB
- **RKE2 and container runtime:** ~3GB
- **Kafka container images:** ~2-3GB
- **Kafka data storage:** Varies based on usage
- **Kubernetes overhead:** ~1-2GB

**Minimum:** 30GB EBS volume
**Recommended:** 100GB EBS volume

**What happens if your disk is too small:**
- Pods get stuck in "Pending" state
- Kubernetes applies a "disk pressure taint" to the node
- New pods cannot schedule
- You'll need to expand the EBS volume mid-deployment (painful)

**How to set disk size during EC2 launch:**
1. In EC2 console, when configuring storage
2. Set root volume size to at least 30GB (100GB recommended)
3. If your instance already exists, you can resize the EBS volume in AWS console, then run:
   ```bash
   sudo growpart /dev/nvme0n1 3
   sudo xfs_growfs /
   ```

### Instance Type Requirements

**Minimum:** t3.medium (2 vCPU, 4GB RAM)
**Recommended:** t3.2xlarge (8 vCPU, 32GB RAM) for running Kafka with 3 brokers + 3 controllers

---

## Part 1: Installing RKE2 on Your EC2 Instance

### What is RKE2?
RKE2 is Rancher's Kubernetes distribution. When you install it, you're essentially setting up a Kubernetes cluster on your server. Think of it as installing the "operating system" for container orchestration.

### Step 1: SSH into Your EC2 Instance

```bash
ssh -i your-key.pem ec2-user@your-ec2-ip
```

**What you're doing:** Just connecting to your server.

**Note:** Username depends on your OS:
- RHEL/Amazon Linux: `ec2-user`
- Ubuntu: `ubuntu`

### Step 2: Decide Your Node Type

RKE2 has two modes:
- **Server mode**: This runs the control plane (the "brain" of Kubernetes)
- **Agent mode**: This is a worker node (does the actual work)

**For a simple setup with one instance, you want SERVER mode** because it includes both the control plane AND can run workloads.

### Step 3: Install RKE2

```bash
curl -sfL https://get.rke2.io | sudo sh -
```

**What's happening:**
- This downloads the RKE2 installation script
- The script detects your OS and installs RKE2 binaries
- It places everything in `/var/lib/rancher/rke2/` and `/etc/rancher/rke2/`

**Why this works:** RKE2 provides an installation script that handles all the complexity for you.

### Step 4: Start RKE2 and Handle the Networking Conflict

First, let's enable and try to start RKE2:

```bash
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
```

**What these commands do:**
- `enable`: Tells your system to start RKE2 automatically when the server boots
- `start`: Starts RKE2 right now

Now check if it's running:

```bash
sudo systemctl status rke2-server.service
```

**If you're on RHEL, CentOS, or Amazon Linux, you'll likely see it failing.** Let's check the logs to confirm:

```bash
sudo journalctl -u rke2-server -f
```

**What you'll probably see:** Errors about `nm-cloud-setup.service` and the service repeatedly restarting.

Press `Ctrl+C` to stop watching the logs.

#### Why Does This Happen?

RKE2 needs complete control over the network configuration to set up Kubernetes networking (called CNI - Container Network Interface). This includes creating virtual network interfaces, routing rules, and IP address management.

On RHEL-based systems, there's a service called `nm-cloud-setup` (NetworkManager Cloud Setup) that also tries to automatically configure networking for cloud instances. If both RKE2 and nm-cloud-setup try to manage networking, they'll conflict and break each other.

RKE2 checks for this conflict before starting. If it finds `nm-cloud-setup` enabled, it refuses to start to prevent network chaos.

#### Fixing the Conflict

We need to disable `nm-cloud-setup` so RKE2 can manage the network:

```bash
# Stop RKE2 (it's already failing, but let's be explicit)
sudo systemctl stop rke2-server.service

# Disable and stop the conflicting NetworkManager cloud setup service
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer
sudo systemctl stop nm-cloud-setup.service nm-cloud-setup.timer
```

**What these commands do:**
- `disable`: Prevents the service from starting on boot
- `stop`: Stops it right now if it's running
- We target both the `.service` and `.timer` because nm-cloud-setup uses a timer to run periodically

Verify it's disabled:
```bash
sudo systemctl status nm-cloud-setup.service
```

You should see "disabled" and "inactive (dead)".

Now start RKE2 again:
```bash
sudo systemctl start rke2-server.service
```

Watch the logs - this time it should start successfully:
```bash
sudo journalctl -u rke2-server -f
```

**What success looks like:**
- Messages about "Starting containerd"
- Messages about "Starting etcd"
- Messages about "Node registered"
- Eventually stabilizes with info messages

This takes **1-2 minutes**. Press `Ctrl+C` once you see things running smoothly.

---

### Step 5: Set Up kubectl Access (And Why It's Not Straightforward)

RKE2 installed `kubectl` (the Kubernetes command-line tool) for you, but you can't use it yet. Let's try:

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
kubectl get nodes
```

**You'll get an error:** `permission denied`

#### Why Does This Happen?

When RKE2 installed and started, it ran as the root user (that's why we used `sudo`). When it created the kubeconfig file at `/etc/rancher/rke2/rke2.yaml`, it created it as root, with root-only permissions.

Your regular user (ec2-user, ubuntu, etc.) doesn't have permission to read root's files. You could use `sudo kubectl` for every command, but that's annoying and not how Kubernetes tools are meant to be used.

#### Why Not Just Change the File Permissions?

You might think: "Why not just `sudo chmod` the file so I can read it?"

The problem is that RKE2 owns that file and may update it. If you mess with its permissions, you could cause issues. Plus, the standard practice in Kubernetes is to have your kubeconfig in `~/.kube/config` - that's where all Kubernetes tools look by default.

#### The Proper Solution: Copy and Own Your Config

We'll copy the kubeconfig to your home directory and make you the owner:

```bash
# Create the standard .kube directory in your home
mkdir -p ~/.kube

# Copy RKE2's kubeconfig to your home directory
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

# Make yourself the owner of the copied file
sudo chown $(id -u):$(id -g) ~/.kube/config

# Secure it so only you can read/write it (Kubernetes best practice)
chmod 600 ~/.kube/config
```

**What each command does:**
- `mkdir -p ~/.kube`: Creates the `.kube` folder in your home directory (the `-p` means "don't error if it exists")
- `sudo cp ...`: Copies the file (we need sudo to read the original file)
- `sudo chown $(id -u):$(id -g) ~/.kube/config`: Changes the owner from root to you
  - `$(id -u)` gets your user ID
  - `$(id -g)` gets your group ID
- `chmod 600`: Sets permissions so only you can read/write this file (security best practice)

Now we need to tell your shell where to find kubectl and the config:

```bash
# Add these to your .bashrc so they're set every time you log in
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc

# Apply the changes to your current session
source ~/.bashrc
```

**What these do:**
- `KUBECONFIG=~/.kube/config`: Tells kubectl where to find your cluster configuration
- `PATH=$PATH:/var/lib/rancher/rke2/bin`: Adds RKE2's bin directory to your PATH so you can run `kubectl` without typing the full path
- `>> ~/.bashrc`: Appends these to your bash profile so they're set automatically when you log in
- `source ~/.bashrc`: Reloads your bash profile so the changes take effect immediately

### Step 6: Verify Your Cluster Works

```bash
kubectl get nodes
```

**What you should see:**
```
NAME          STATUS   ROLES                       AGE   VERSION
your-node     Ready    control-plane,etcd,master   2m    v1.28.x+rke2
```

**What this means:**
- Your Kubernetes cluster is running
- Your node is "Ready" (healthy and working)
- It's acting as both control plane and worker

---

## Part 2: Understanding Kubernetes Basics (Quick Primer)

Before deploying Kafka, understand these concepts:

### Namespace
Think of it like a folder that organizes your applications. Keeps things tidy.

### Pod
A pod is the smallest unit in Kubernetes. It's a wrapper around your container(s). When we deploy Kafka, it runs in pods.

### Service
Services give pods stable networking. Pods can die and restart with new IPs, but services provide a consistent way to reach them.

### Helm
Helm is like a package manager for Kubernetes (think apt/yum for Linux). It bundles all the YAML configs needed to deploy complex apps like Kafka.

---

## Part 3: Deploying Confluent Kafka with KRaft Mode

### What is KRaft Mode?

**ZooKeeper was deprecated in November 2025.** KRaft (Kafka Raft) is the new architecture that removes the ZooKeeper dependency. Instead of ZooKeeper coordinating the cluster, Kafka now has dedicated controller nodes that handle metadata using the Raft consensus protocol.

**Why this matters:**
- Simpler architecture (fewer moving parts)
- Better performance and scalability
- Official Confluent direction going forward

### Step 1: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**What's happening:** Installing Helm 3, required for deploying Confluent for Kubernetes.

### Step 2: Set Up Storage Class

RKE2 includes a storage provisioner called `local-path` that creates storage volumes on the node's disk. We need to set it as the default:

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Why this matters:**
- Kafka needs persistent storage for its data
- Without a default storage class, Kafka pods will remain "Pending"
- This tells Kubernetes where to create storage volumes

Verify it worked:
```bash
kubectl get storageclass
```

You should see `local-path` marked as `(default)`.

### Step 3: Download Confluent for Kubernetes

**Important:** Confluent for Kubernetes (CFK) is NOT in public Helm repositories. You must download it as a bundle:

```bash
# Download the official CFK bundle
curl -O https://packages.confluent.io/bundle/cfk/confluent-for-kubernetes-3.1.0.tar.gz

# Extract it
tar -xvf confluent-for-kubernetes-3.1.0.tar.gz
```

**What's happening:**
- Downloading ~218MB bundle containing the operator and examples
- Extracting to a directory with timestamp in the name

### Step 4: Install the Confluent for Kubernetes Operator

```bash
# Install from the extracted bundle
helm upgrade --install confluent-operator \
  ./confluent-for-kubernetes-3.1.0-*/helm/confluent-for-kubernetes \
  --set kRaftEnabled=true \
  --namespace confluent \
  --create-namespace
```

**What's happening:**
- Installing the CFK operator (the "brain" that manages Kafka)
- Enabling KRaft mode support
- Creating a new namespace called `confluent`

**Why the operator pattern:**
- The operator watches for Kafka resource definitions
- Automatically creates and manages all necessary Kubernetes resources
- Handles upgrades, scaling, and recovery

Verify the operator is running:
```bash
kubectl get pods -n confluent
```

You should see `confluent-operator-xxxxx` in "Running" status.

### Step 5: Deploy KRaft Controllers and Kafka Brokers

Create a configuration file for your Kafka cluster:

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

**What each part means:**

**KRaftController:**
- `replicas: 3`: Creates 3 controller nodes (odd number required for Raft consensus)
- `application: confluentinc/cp-server:7.9.1`: Official Confluent Platform image
- `dataVolumeCapacity: 10Gi`: Each controller gets 10GB of storage
- Controllers manage cluster metadata, handle leader elections, and coordinate the cluster

**Kafka:**
- `replicas: 3`: Creates 3 Kafka broker nodes
- Brokers handle actual message storage and client connections
- `dependencies.kRaftController`: Links brokers to the controllers

**Why 3 replicas:**
- Provides high availability
- Allows 1 node to fail without data loss
- Enables proper replication and failover

Apply the configuration:
```bash
kubectl apply -f confluent-kafka-kraft.yaml
```

### Step 6: Watch the Deployment

```bash
kubectl get pods -n confluent -w
```

**What you'll see (in this order):**
1. **kraftcontroller-0, -1, -2** pods: Init → Running (2-3 minutes)
2. **kafka-0, -1, -2** pods: Init → Running (2-3 minutes after controllers are ready)

**What's happening:**
- Controllers start first and form a Raft quorum
- Once controllers are stable, Kafka brokers start
- Each pod downloads ~2GB of container images
- Init containers configure the pods before main containers start

Press `Ctrl+C` once all pods show "1/1 Running".

---

## Part 4: Verify Kafka is Working

### Step 1: Check All Pods are Running

```bash
kubectl get pods -n confluent
```

**You should see:**
- `confluent-operator-xxxxx`: 1/1 Running
- `kraftcontroller-0, -1, -2`: 1/1 Running
- `kafka-0, -1, -2`: 1/1 Running

**Total: 7 pods, all in "Running" status.**

### Step 2: Create a Test Client Pod

Instead of exec'ing into Kafka pods (which can interfere with the cluster), create a dedicated client pod:

```bash
kubectl run kafka-client -n confluent \
  --image=confluentinc/cp-server:7.9.1 \
  --restart=Never \
  --command -- sleep infinity
```

**What's happening:**
- Creating a pod with Kafka tools installed
- `sleep infinity` keeps it running so we can exec into it
- This is the proper way to interact with Kafka

Wait for it to be ready:
```bash
kubectl wait --for=condition=ready pod/kafka-client -n confluent --timeout=60s
```

### Step 3: Create a Test Topic

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 3
```

**What's happening:**
- `kafka-topics`: Kafka's command-line tool for managing topics
- `--bootstrap-server kafka.confluent.svc.cluster.local:9092`: Connect to the Kafka cluster
  - `kafka`: Service name
  - `confluent`: Namespace
  - `svc.cluster.local`: Kubernetes DNS suffix
  - `9092`: Kafka's default port
- `--partitions 3`: Split the topic into 3 partitions for parallelism
- `--replication-factor 3`: Each partition is replicated across all 3 brokers

**What's a topic?** Think of it like a category or channel where messages are published.

**Why 3 partitions and 3 replicas?**
- Partitions enable parallel processing
- Replication ensures no data loss if a broker fails

### Step 4: Verify the Topic

```bash
kubectl exec -n confluent kafka-client -- kafka-topics \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --describe \
  --topic test-topic
```

**You should see:**
- Topic details showing 3 partitions
- Each partition replicated across 3 brokers
- Leader and replica information

### Step 5: Send Test Messages (Producer)

```bash
kubectl exec -n confluent kafka-client -- bash -c \
  "echo 'Hello from Confluent Kafka on RKE2!' | kafka-console-producer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic"
```

**What's happening:**
- Using `kafka-console-producer` to send a message
- The message goes to one of the 3 partitions
- Kafka replicates it to all 3 brokers

### Step 6: Read the Message (Consumer)

```bash
kubectl exec -n confluent kafka-client -- kafka-console-consumer \
  --bootstrap-server kafka.confluent.svc.cluster.local:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 1
```

**What you should see:** Your message "Hello from Confluent Kafka on RKE2!" printed out.

**If you see this, congratulations! Kafka is working!** 🎉

---

## Part 5: Understanding How to Connect to Kafka

### From Inside the Cluster (Other Pods)

Applications running in Kubernetes can connect to Kafka using the service name:
```
kafka.confluent.svc.cluster.local:9092
```

**Breaking this down:**
- `kafka`: Service name (created by CFK operator)
- `confluent`: Namespace
- `svc.cluster.local`: Kubernetes DNS suffix
- `9092`: Kafka's default port

Your application configuration would look like:
```properties
bootstrap.servers=kafka.confluent.svc.cluster.local:9092
```

### From Outside the Cluster (Your Applications)

For external access, you have a few options:

**Option 1: NodePort (Simple, not production-grade)**
```bash
kubectl expose service kafka \
  --name=kafka-external \
  --namespace=confluent \
  --type=NodePort \
  --port=9092
```

Get the assigned NodePort:
```bash
kubectl get svc kafka-external -n confluent
```

Connect using: `<your-ec2-public-ip>:<nodeport>`

**Important:**
- Make sure your EC2 security group allows traffic on that NodePort (30000-32767 range)
- NodePort is fine for testing but not recommended for production

**Option 2: LoadBalancer (AWS specific, better for production)**
```bash
kubectl expose service kafka \
  --name=kafka-lb \
  --namespace=confluent \
  --type=LoadBalancer \
  --port=9092
```

This creates an AWS ELB. Get the address:
```bash
kubectl get svc kafka-lb -n confluent
```

**Option 3: Port Forward (Development/Testing only)**
```bash
kubectl port-forward -n confluent svc/kafka 9092:9092
```

Then connect to `localhost:9092` from your local machine. Press Ctrl+C to stop.

---

## Part 6: Common Operations

### See What's Running
```bash
kubectl get pods -n confluent
kubectl get svc -n confluent
kubectl get pvc -n confluent  # See persistent storage
```

### Check Pod Logs
```bash
# Kafka broker logs
kubectl logs -n confluent kafka-0

# Controller logs
kubectl logs -n confluent kraftcontroller-0

# Operator logs
kubectl logs -n confluent deployment/confluent-operator
```

### Get Shell Access to Client Pod
```bash
kubectl exec -it -n confluent kafka-client -- bash
```

**Once inside, you can run any Kafka command directly.**

### Scale the Cluster
```bash
# Edit the Kafka resource to change replicas
kubectl edit kafka kafka -n confluent

# Or update your YAML and reapply
# Change replicas: 3 to replicas: 5
kubectl apply -f confluent-kafka-kraft.yaml
```

### Delete Everything (Start Over)
```bash
# Delete Kafka and controllers
kubectl delete -f confluent-kafka-kraft.yaml

# Delete the operator
helm uninstall confluent-operator -n confluent

# Delete the namespace (this removes PVCs too)
kubectl delete namespace confluent
```

---

## Troubleshooting: Common Issues and Solutions

### Issue 1: Pods Stuck in "Pending" with Disk Pressure

**Symptoms:**
```bash
kubectl get pods -n confluent
# Shows pods in "Pending" status

kubectl describe node
# Shows: Taints: node.kubernetes.io/disk-pressure:NoSchedule
```

**What's happening:**
- Kubernetes detected high disk usage (>85%)
- Applied a "disk pressure taint" to protect the node
- Scheduler refuses to place new pods on tainted nodes

**Root causes:**
- EC2 disk too small (10GB is insufficient)
- Leftover PersistentVolumeClaims (PVCs) from failed deployments
- Old container images consuming space

**Solution 1: Check disk usage**
```bash
df -h /
```

If disk is >80% full, you need to expand it.

**Solution 2: Expand EBS volume (if disk is too small)**
```bash
# In AWS Console:
# 1. EC2 → Volumes → Select your volume → Actions → Modify Volume
# 2. Change size to 100GB → Modify

# Then on the EC2 instance:
sudo growpart /dev/nvme0n1 3
sudo xfs_growfs /
df -h /  # Verify new size
```

**Solution 3: Clean up leftover PVCs**
```bash
# List PVCs
kubectl get pvc --all-namespaces

# Delete old PVCs (if you see any from deleted deployments)
kubectl delete pvc <pvc-name> -n <namespace>

# Check PersistentVolumes too
kubectl get pv
# If any show "Released" status, delete them:
kubectl delete pv <pv-name>
```

**Solution 4: Remove the taint manually (after fixing disk space)**
```bash
kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule-
```

**Solution 5: Prune old container images**
```bash
sudo /var/lib/rancher/rke2/bin/crictl \
  --runtime-endpoint unix:///run/k3s/containerd/containerd.sock \
  rmi --prune
```

### Issue 2: kubectl Not Found After Re-login

**Symptom:** After logging out and back into EC2, `kubectl` command not found.

**What's happening:**
- `.bashrc` contains your exports, but `.bash_profile` runs first
- `.bash_profile` might be overwriting your PATH

**Solution:** Add exports to `.bash_profile` instead:
```bash
# Edit .bash_profile
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bash_profile
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bash_profile
source ~/.bash_profile
```

### Issue 3: Pods Stuck in "ContainerCreating"

**Be patient.** Kubernetes is downloading container images. Each Kafka image is ~2GB.

**To verify progress:**
```bash
kubectl describe pod <pod-name> -n confluent
# Look for "pulling image" or "image pull" messages
```

**Expected time:** 3-5 minutes per pod depending on network speed.

### Issue 4: No Storage Class Available

**Symptom:** Pods stuck in Pending with message "unbound immediate PersistentVolumeClaims"

**Solution:**
```bash
# Set local-path as default storage class
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# Should show: local-path (default)
```

### Issue 5: Can't Connect to Kafka from Outside the Cluster

**Check these in order:**
1. **Is your EC2 security group configured?**
   - Allow traffic on NodePort range (30000-32767)
   - Or LoadBalancer port (9092)

2. **Did you expose the service?**
   ```bash
   kubectl get svc -n confluent
   # Should see kafka-external or kafka-lb
   ```

3. **Are you using the correct address?**
   - NodePort: `<ec2-public-ip>:<nodeport>` (NOT 9092)
   - LoadBalancer: Use the EXTERNAL-IP from `kubectl get svc`

4. **Is Kafka actually running?**
   ```bash
   kubectl get pods -n confluent
   # All kafka-* pods should be 1/1 Running
   ```

### Issue 6: RKE2 Won't Start on RHEL/CentOS

**Symptom:** `sudo journalctl -u rke2-server` shows errors about `nm-cloud-setup.service`

**Solution:** See Part 1, Step 4 - disable nm-cloud-setup service.

### Issue 7: "Evicted" Pods

**Symptom:** Pods show status "Evicted"

**What happened:** Kubernetes evicted pods due to resource pressure (disk/memory).

**Solution:**
```bash
# Delete evicted pods (they're dead anyway)
kubectl get pods -n confluent | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n confluent

# Fix the underlying cause (usually disk pressure - see Issue 1)
```

---

## Key Files and Locations

**RKE2:**
- **Config**: `/etc/rancher/rke2/config.yaml`
- **Kubeconfig**: `/etc/rancher/rke2/rke2.yaml` (copy to `~/.kube/config`)
- **Data**: `/var/lib/rancher/rke2/`
- **Logs**: `sudo journalctl -u rke2-server`
- **Binaries**: `/var/lib/rancher/rke2/bin/`

**Kafka:**
- **Configuration file**: `confluent-kafka-kraft.yaml` (in your home directory)
- **Operator helm chart**: `./confluent-for-kubernetes-3.1.0-*/helm/confluent-for-kubernetes`
- **Persistent data**: `/opt/local-path-provisioner/pvc-*/` (managed by local-path provisioner)

---

## What You've Learned

1. ✅ How to properly size an EC2 instance for Kubernetes + Kafka
2. ✅ How to install and configure RKE2 on RHEL/CentOS
3. ✅ How to resolve the nm-cloud-setup networking conflict
4. ✅ How to configure kubectl access with proper permissions
5. ✅ How to deploy Confluent for Kubernetes operator
6. ✅ How KRaft mode works (ZooKeeper-free Kafka)
7. ✅ How to troubleshoot disk pressure and PVC issues
8. ✅ How to test Kafka (create topics, produce/consume messages)
9. ✅ How to connect to Kafka from inside and outside the cluster

---

## Important Takeaways

### Disk Space Matters
- 10GB is too small for Kubernetes + Kafka
- Always start with 30GB minimum, 100GB recommended
- Monitor disk usage: `df -h /`
- Kubernetes will taint nodes at 85% disk usage

### PVCs Persist After Pod Deletion
- Deleting pods doesn't delete their storage
- Always check for leftover PVCs: `kubectl get pvc --all-namespaces`
- Clean up failed deployments completely

### KRaft vs ZooKeeper
- KRaft is the future (ZooKeeper deprecated November 2025)
- Simpler architecture, better performance
- Use official Confluent images: `confluentinc/cp-server:7.9.1`

### Storage Class Setup
- RKE2 includes `local-path` provisioner
- Must be set as default: `kubectl patch storageclass...`
- Without default storage class, pods stay Pending

---

## Next Steps

### Improve Reliability
- Add more EC2 instances as worker nodes for true high availability
- Set up monitoring with Prometheus + Grafana
- Configure automated backups of Kafka data

### Secure Your Cluster
- Enable TLS/SSL for Kafka brokers
- Set up authentication (SASL/SCRAM or mTLS)
- Configure Kubernetes RBAC
- Use network policies to restrict pod communication

### Optimize Performance
- Tune Kafka broker configurations for your workload
- Adjust resource requests/limits based on usage
- Consider using EBS volumes instead of local-path for better durability
- Enable compression for better throughput

### Add Monitoring
- Deploy Confluent Control Center for Kafka monitoring
- Set up Prometheus to scrape Kafka metrics
- Create Grafana dashboards for visualization
- Configure alerts for disk space, lag, and errors

---

**You now have a production-ready Kubernetes cluster running Confluent Kafka with KRaft mode!** 🚀
