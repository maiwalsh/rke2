# Simple Walkthrough: RKE2 on EC2 with Confluent Kafka

This is a straightforward guide that explains **what** you're doing and **why** at each step.

---

## Understanding the Goal

You want to:
1. Install RKE2 (a Kubernetes distribution) on your EC2 instance
2. Use that Kubernetes cluster to run Confluent Kafka in a pod
3. Understand how everything connects

---

## Part 1: Installing RKE2 on Your EC2 Instance

### What is RKE2?
RKE2 is Rancher's Kubernetes distribution. When you install it, you're essentially setting up a Kubernetes cluster on your server. Think of it as installing the "operating system" for container orchestration.

### Step 1: SSH into Your EC2 Instance

```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

**What you're doing:** Just connecting to your server.

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

## Part 3: Deploying Confluent Kafka

### Step 1: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**What's happening:** Installing Helm, which we'll use to deploy Kafka easily.

### Step 2: Add the Confluent Helm Repository

```bash
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
```

**What's happening:**
- Adding Confluent's Helm chart repository (like adding a package source)
- Updating the list of available charts

**Think of it like:** Adding a new software repository to your package manager.

### Step 3: Create a Namespace for Kafka

```bash
kubectl create namespace kafka
```

**What's happening:** Creating a dedicated "folder" for Kafka and its components.

**Why:** Keeps Kafka isolated from other apps you might deploy later.

### Step 4: Understand What You're Deploying

Confluent Kafka needs several components:

1. **Zookeeper** (3 pods): Coordinates the Kafka cluster
2. **Kafka Brokers** (3 pods): The actual Kafka servers
3. **Schema Registry** (optional): Manages data schemas
4. **Kafka REST** (optional): REST API for Kafka

**For a basic setup, you need Zookeeper + Kafka Brokers.**

### Step 5: Deploy Kafka with Helm

**Simple deployment (uses defaults):**
```bash
helm install kafka confluentinc/cp-helm-charts \
  --namespace kafka \
  --set cp-zookeeper.servers=1 \
  --set cp-kafka.brokers=1
```

**What's happening:**
- `helm install kafka`: Installing a release named "kafka"
- `--namespace kafka`: Put it in the kafka namespace
- `--set cp-zookeeper.servers=1`: Run 1 Zookeeper instance (simple setup)
- `--set cp-kafka.brokers=1`: Run 1 Kafka broker (simple setup)

**Note:** For production, you'd use 3 of each for high availability, but 1 is fine for learning.

### Step 6: Watch It Deploy

```bash
kubectl get pods -n kafka -w
```

**What you'll see:**
- Pods starting up with status "ContainerCreating"
- Then "Running" after a few minutes
- Press `Ctrl+C` to stop watching

**What's happening:**
- Kubernetes is downloading container images
- Starting Zookeeper first (Kafka needs it)
- Then starting Kafka brokers

This typically takes **2-5 minutes** depending on your instance's network speed.

---

## Part 4: Verify Kafka is Working

### Step 1: Check All Pods are Running

```bash
kubectl get pods -n kafka
```

**You should see all pods with STATUS = "Running".**

### Step 2: Check Services

```bash
kubectl get svc -n kafka
```

**What you'll see:** Services that provide networking to your Kafka pods.

Look for:
- `kafka-cp-kafka`: The Kafka broker service
- `kafka-cp-zookeeper`: The Zookeeper service

### Step 3: Test Kafka - Create a Topic

First, get the Kafka pod name:
```bash
kubectl get pods -n kafka | grep cp-kafka
```

You'll see something like `kafka-cp-kafka-0`. Use that name in the next command:

```bash
kubectl exec -n kafka kafka-cp-kafka-0 -- kafka-topics \
  --bootstrap-server localhost:9092 \
  --create \
  --topic test-topic \
  --partitions 1 \
  --replication-factor 1
```

**What's happening:**
- `kubectl exec`: Run a command inside a pod
- `kafka-topics`: Kafka's tool for managing topics
- `--bootstrap-server localhost:9092`: Connect to Kafka (locally within the pod)
- `--create --topic test-topic`: Create a new topic named "test-topic"

**What's a topic?** Think of it like a category or channel where messages are published.

### Step 4: List Topics to Verify

```bash
kubectl exec -n kafka kafka-cp-kafka-0 -- kafka-topics \
  --bootstrap-server localhost:9092 \
  --list
```

**You should see:** `test-topic` in the output.

### Step 5: Send a Test Message (Producer)

```bash
kubectl exec -n kafka kafka-cp-kafka-0 -- bash -c \
  "echo 'Hello from Kafka!' | kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic test-topic"
```

**What's happening:** Sending the message "Hello from Kafka!" to the test-topic.

### Step 6: Read the Message (Consumer)

```bash
kubectl exec -n kafka kafka-cp-kafka-0 -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning \
  --max-messages 1
```

**What you should see:** Your message "Hello from Kafka!" printed out.

**If you see this, congratulations! Kafka is working!** 🎉

---

## Part 5: Understanding How to Connect to Kafka

### From Inside the Cluster (Other Pods)

Applications running in Kubernetes can connect to Kafka using the service name:
```
kafka-cp-kafka.kafka.svc.cluster.local:9092
```

**Breaking this down:**
- `kafka-cp-kafka`: Service name
- `kafka`: Namespace
- `svc.cluster.local`: Kubernetes DNS suffix
- `9092`: Kafka's default port

### From Outside the Cluster (Your Applications)

You have a few options:

**Option 1: NodePort (Simple, not production-grade)**
```bash
kubectl expose service kafka-cp-kafka \
  --name=kafka-external \
  --namespace=kafka \
  --type=NodePort \
  --port=9092
```

Then get the port:
```bash
kubectl get svc kafka-external -n kafka
```

Connect using: `<your-ec2-public-ip>:<nodeport>`

**Important:** Make sure your EC2 security group allows traffic on that NodePort (30000-32767 range).

**Option 2: LoadBalancer (AWS specific)**
```bash
kubectl expose service kafka-cp-kafka \
  --name=kafka-lb \
  --namespace=kafka \
  --type=LoadBalancer \
  --port=9092
```

This creates an AWS ELB. Get the address:
```bash
kubectl get svc kafka-lb -n kafka
```

---

## Part 6: Common Operations

### See What's Running
```bash
kubectl get pods -n kafka
kubectl get svc -n kafka
```

### Check Pod Logs
```bash
kubectl logs -n kafka <pod-name>
```

### Get Shell Access to Kafka Pod
```bash
kubectl exec -it -n kafka kafka-cp-kafka-0 -- bash
```

**Once inside, you can run any Kafka command directly.**

### Delete Everything (Start Over)
```bash
helm uninstall kafka -n kafka
kubectl delete namespace kafka
```

---

## Troubleshooting Tips

### Pods Stuck in "Pending"
**Check:** `kubectl describe pod <pod-name> -n kafka`

**Common cause:** Not enough resources. Your EC2 instance might be too small. Kafka needs at least a t3.medium instance.

### Pods Stuck in "ContainerCreating"
**Be patient.** Kubernetes is downloading container images from the internet. This can take 3-5 minutes depending on your network speed.

**To verify it's actually downloading:** `kubectl describe pod <pod-name> -n kafka` and look for "pulling image" messages.

### Can't Connect to Kafka from Outside the Cluster
**Check these in order:**
1. Is your EC2 security group configured to allow traffic on the NodePort? (Check port range 30000-32767)
2. Did you expose the service as NodePort or LoadBalancer? Run `kubectl get svc -n kafka` to verify
3. Are you using the correct IP:port? NodePort uses `<ec2-public-ip>:<nodeport>`, not 9092
4. Is the Kafka pod actually running? `kubectl get pods -n kafka`

---

## Key Files and Locations

- **RKE2 config**: `/etc/rancher/rke2/config.yaml`
- **Kubeconfig**: `/etc/rancher/rke2/rke2.yaml`
- **RKE2 data**: `/var/lib/rancher/rke2/`
- **RKE2 logs**: `sudo journalctl -u rke2-server`

---

## What You've Learned

1. ✅ How to install RKE2 on an EC2 instance
2. ✅ How RKE2 creates a Kubernetes cluster
3. ✅ How to use kubectl to interact with Kubernetes
4. ✅ How to deploy applications using Helm
5. ✅ How Kafka runs in Kubernetes pods
6. ✅ How to test Kafka (create topics, produce/consume messages)
7. ✅ How to expose services for external access

---

## Next Steps

- **Scale up:** Add more Kafka brokers (`--set cp-kafka.brokers=3`)
- **Persistence:** Add storage so data survives pod restarts
- **Monitoring:** Deploy Prometheus/Grafana
- **Security:** Set up authentication and encryption
- **Production:** Add more EC2 instances as worker nodes

---

**You now have a working Kubernetes cluster with Kafka running on it!** 🚀
