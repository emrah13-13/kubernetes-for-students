# Chapter 0: Setup Your Playground

Before you play with Kubernetes, you need a sandbox first. Can't learn to drive without a car, right?

## What You Need

### Hardware
- **RAM**: 8GB minimum, 16GB recommended
- **CPU**: Dual-core minimum, quad-core better
- **Storage**: 20GB free space
- **OS**: Windows 10/11, macOS, or Linux (any recent distro)

Why these specs? Kubernetes clusters consume resources. You'll run multiple containers simultaneously, plus the control plane coordinating everything.

## Required Tools

### 1. Docker Desktop

**Why do you need it?**  
Kubernetes orchestrates containers. You need a container runtime before orchestrating anything.

**Install:**

**Windows/macOS:**
- Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
- Install, restart computer if prompted
- Open Docker Desktop, wait until status shows "running"

**Linux:**
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group so you don't need sudo
sudo usermod -aG docker $USER
# Logout and login again
```

**Verify:**
```bash
docker --version
# Output: Docker version 24.x.x or newer

docker run hello-world
# If you see "Hello from Docker!" it's successful
```

### 2. kubectl - The Kubernetes CLI

**Why do you need it?**  
This is the command-line tool for communicating with Kubernetes clusters. Everything you do in K8s goes through kubectl.

**Install:**

**Windows (PowerShell as Admin):**
```powershell
# Via Chocolatey (recommended)
choco install kubernetes-cli

# Or manual download
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe ke folder yang ada di PATH
```

**macOS:**
```bash
# Via Homebrew
brew install kubectl

# Or manual
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Verify:**
```bash
kubectl version --client
# Output: Client Version: v1.28.x
```

### 3. Minikube - Local Kubernetes Cluster

**Why do you need it?**  
Production Kubernetes clusters are expensive and complex to set up. Minikube creates a local cluster on your laptop, perfect for learning.

**Install:**

**Windows:**
```powershell
# Via Chocolatey
choco install minikube

# Or manual download
# https://minikube.sigs.k8s.io/docs/start/
```

**macOS:**
```bash
brew install minikube
```

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Verify:**
```bash
minikube version
# Output: minikube version: v1.31.x
```

## First Cluster - The Moment of Truth

Now you have all the tools. Time to spin up your first cluster:

```bash
# Start minikube cluster
minikube start

# Expected output:
# - Downloading Kubernetes components
# - Creating VM or container
# - Configuring environment
# - Done! kubectl is configured
```

**Troubleshooting Start Issues:**

**Error: "Insufficient memory"**
```bash
# Start with smaller resources
minikube start --memory=4096 --cpus=2
```

**Error: "Driver not found"**
```bash
# Specify driver explicitly
minikube start --driver=docker
# or
minikube start --driver=virtualbox
```

**Error on Windows: "Hyper-V required"**
- Enable Hyper-V in Windows Features
- Or use Docker driver: `minikube start --driver=docker`

## Verify Everything Works

After minikube starts successfully, try this:

```bash
# Check cluster status
minikube status

# Expected output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured
```

```bash
# Check nodes
kubectl get nodes

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.28.x
```

If you see output like that, congratulations. You have a Kubernetes cluster running on your laptop.

## Understanding What Just Happened

When you ran `minikube start`, here's what happened:

1. **VM/Container Created** - Minikube creates a lightweight VM or container to run the cluster
2. **Kubernetes Components Started** - Control plane components (API server, scheduler, controller manager) are started
3. **kubectl Configured** - Config file created at `~/.kube/config` so kubectl knows how to connect
4. **Node Ready** - One node (named "minikube") is ready to accept workloads

Analogy: You just set up a personal data center on your laptop. Small, yes, only one "server", but it's a fully functional Kubernetes cluster.

## Useful Minikube Commands

```bash
# Stop cluster (ga delete data)
minikube stop

# Start lagi cluster yang udah ada
minikube start

# Delete cluster completely
minikube delete

# SSH into the minikube VM
minikube ssh

# Open Kubernetes dashboard (web UI)
minikube dashboard

# Check minikube IP
minikube ip
```

## Setting Up Your Workspace

Create a folder to store YAML files and experiments:

```bash
mkdir -p ~/k8s-learning
cd ~/k8s-learning

# Test by creating a simple namespace
kubectl create namespace test
kubectl get namespaces
```

## Optional but Helpful: k9s

k9s is a terminal UI for Kubernetes. More user-friendly than pure kubectl commands.

**Install:**
```bash
# macOS
brew install k9s

# Linux
curl -sS https://webinstall.dev/k9s | bash

# Windows
choco install k9s
```

**Usage:**
```bash
k9s
# Press '0' to show all namespaces
# Use arrow keys to navigate
# Press 'd' to describe resource
# Press 'l' to see logs
# Press ':q' to quit
```

## Checklist

Before moving to Chapter 1, make sure you can:

- [ ] `docker --version` shows installed version
- [ ] `kubectl version --client` shows installed version
- [ ] `minikube version` shows installed version
- [ ] `minikube start` succeeds without errors
- [ ] `kubectl get nodes` shows one node in Ready status
- [ ] `minikube dashboard` opens web UI (optional but good for visual learners)

## Common Setup Issues

### Issue: Minikube stuck at "Starting control plane"

**Solution:**
```bash
minikube delete
minikube start --driver=docker --alsologtostderr
# Check logs for specific errors
```

### Issue: kubectl command not found

**Solution:**
Make sure kubectl is in PATH. Test:
```bash
which kubectl  # macOS/Linux
where kubectl  # Windows

# If not found, re-install and add to PATH
```

### Issue: Permission denied errors (Linux)

**Solution:**
```bash
# Add user to docker group
sudo usermod -aG docker $USER
# Logout and login again
```

## Next Steps

Setup complete? Time to actually use Kubernetes.

**Go to**: [Chapter 1: First Contact](../01-first-contact/)

In the next chapter, you'll:
- Deploy your first application to the cluster
- Understand what a Pod is
- Interact with running containers in Kubernetes

---

**Estimated time**: 30-60 minutes (including downloads)  
**Difficulty**: Easy  
**Can you skip this?**: No. Everything else depends on this.
