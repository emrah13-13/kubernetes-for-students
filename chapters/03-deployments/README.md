# Chapter 3: Deployments - The Copy Machine

You learned that bare pods die and don't come back. So how do production apps stay running? Enter Deployments.

## The Office Copy Machine Analogy

Imagine you need to print 100 copies of a document. You don't:
- Print one, wait, print another, wait... (too slow)
- Worry if the printer jams (it auto-recovers)
- Manually replace paper or toner (it manages itself)

You just tell the copier: "I need 100 copies of this document, keep them coming."

**Deployments** are Kubernetes' copy machine. You declare:
- What image to run (the document)
- How many copies you want (replicas)
- How to update them when needed (rolling update strategy)

Kubernetes ensures that many copies are always running, replacing failed ones automatically.

## The Reality: What is a Deployment?

A **Deployment** is a controller that manages a ReplicaSet, which manages Pods.

Hierarchy:
```
Deployment
  └── ReplicaSet
        ├── Pod 1
        ├── Pod 2
        └── Pod 3
```

**Deployment** provides:
- **Desired state management** - "I want 3 replicas"
- **Self-healing** - Pod dies? Deployment creates a new one
- **Scaling** - Change replica count on the fly
- **Rolling updates** - Update app version with zero downtime
- **Rollback** - Oops, new version breaks? Roll back

## Experiment 1: Create Your First Deployment

Let's deploy nginx with 3 replicas.

**Create file: `nginx-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3  # We want 3 copies
  selector:
    matchLabels:
      app: nginx  # Which pods does this deployment manage?
  template:  # Pod template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Deploy it:**
```bash
kubectl apply -f nginx-deployment.yaml

# Check deployment
kubectl get deployments

# Output:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           20s
```

**Check the pods it created:**
```bash
kubectl get pods

# Output: You'll see 3 pods with names like:
# nginx-deployment-7d64c8d7f5-abcde
# nginx-deployment-7d64c8d7f5-fghij
# nginx-deployment-7d64c8d7f5-klmno
```

Notice the pod names? They're auto-generated: `<deployment-name>-<replicaset-hash>-<random-id>`

## Experiment 2: Self-Healing in Action

Let's kill a pod and watch Kubernetes bring it back.

```bash
# Get pods
kubectl get pods

# Delete one pod (replace with actual pod name)
kubectl delete pod nginx-deployment-7d64c8d7f5-abcde

# Immediately check pods
kubectl get pods

# You'll see:
# - The pod you deleted is Terminating
# - A NEW pod is already being created
# - Total count goes back to 3
```

**What happened?**
1. You deleted a pod
2. Deployment noticed: "Hey, I wanted 3, now there's only 2"
3. Deployment told ReplicaSet: "Create another pod"
4. ReplicaSet created a new pod
5. Desired state restored

This is **self-healing**. In production, if a pod crashes, Deployment auto-recovers.

## Experiment 3: Scaling

Need more replicas? Just change the number.

**Option 1: Edit YAML and re-apply**
```yaml
# Change replicas: 3 to replicas: 5
spec:
  replicas: 5
```

```bash
kubectl apply -f nginx-deployment.yaml

kubectl get pods
# Now you see 5 pods
```

**Option 2: Scale via kubectl**
```bash
# Scale to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

kubectl get pods
# 5 pods running

# Scale down to 2
kubectl scale deployment nginx-deployment --replicas=2

kubectl get pods
# Watch as 3 pods terminate, leaving 2
```

Scaling is instant. Kubernetes just adjusts pod count to match desired state.

## Experiment 4: Rolling Updates

Time to update the nginx version without downtime.

**Current state:** Running `nginx:1.25`  
**Goal:** Update to `nginx:1.26` with zero downtime

**Edit YAML:**
```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.26  # Changed from 1.25 to 1.26
```

**Apply the update:**
```bash
kubectl apply -f nginx-deployment.yaml

# Watch the rollout
kubectl rollout status deployment nginx-deployment
```

**Watch pods during update:**
```bash
kubectl get pods --watch
```

You'll see:
1. New pod created with nginx:1.26
2. Once it's ready, old pod terminated
3. Another new pod created
4. Another old pod terminated
5. Repeat until all pods are updated

This is **rolling update**. At no point are all pods down. Traffic keeps flowing.

**Check rollout history:**
```bash
kubectl rollout history deployment nginx-deployment

# Output shows revision history
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

## Experiment 5: Rollback

New version has a bug. Roll back instantly.

```bash
# Rollback to previous version
kubectl rollout undo deployment nginx-deployment

# Check status
kubectl rollout status deployment nginx-deployment

# Verify image
kubectl describe deployment nginx-deployment | grep Image
# Should show nginx:1.25 again
```

**Rollback to specific revision:**
```bash
# See history
kubectl rollout history deployment nginx-deployment

# Rollback to revision 1
kubectl rollout undo deployment nginx-deployment --to-revision=1
```

## Understanding Deployment Strategy

There are 2 main strategies:

### 1. RollingUpdate (Default)

Gradually replace old pods with new ones.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max extra pods during update
      maxUnavailable: 0  # Max unavailable pods during update
```

**maxSurge: 1** = During update, can have 4 pods instead of 3 temporarily  
**maxUnavailable: 0** = Never have less than 3 pods running

This ensures zero downtime but uses more resources temporarily.

### 2. Recreate

Kill all old pods, then create new ones.

```yaml
spec:
  strategy:
    type: Recreate
```

**Use when:**
- App can't run multiple versions simultaneously
- Okay with brief downtime
- Want to save resources

## The Breakdown: Deployment YAML

```yaml
apiVersion: apps/v1      # Deployment is in apps/v1 API group
kind: Deployment

metadata:
  name: nginx-deployment
  labels:
    app: nginx          # Labels for the deployment itself

spec:
  replicas: 3           # How many pod copies

  selector:
    matchLabels:
      app: nginx        # MUST match template.metadata.labels

  template:             # Pod template
    metadata:
      labels:
        app: nginx      # Labels for the pods
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Critical:** `selector.matchLabels` MUST match `template.metadata.labels`. This is how Deployment knows which pods it manages.

## Common Mistakes

### Mistake 1: Selector Mismatch

```yaml
spec:
  selector:
    matchLabels:
      app: web      # Says "web"
  template:
    metadata:
      labels:
        app: nginx  # But pods labeled "nginx"
```

**Result:** Deployment won't work. Selector can't find matching pods.

### Mistake 2: Not Enough Resources

If your cluster doesn't have resources for replicas, pods stay Pending.

```bash
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-deployment-xxx     0/1     Pending   0          2m
```

**Check why:**
```bash
kubectl describe pod nginx-deployment-xxx
# Events: Insufficient memory or CPU
```

**Fix:** Reduce replicas or add more nodes.

### Mistake 3: Image Pull Errors

```yaml
image: ngnix:1.25  # Typo: ngnix instead of nginx
```

```bash
kubectl get pods
# STATUS: ImagePullBackOff
```

**Fix:** Correct the image name.

## Debugging Deployments

**Check deployment status:**
```bash
kubectl get deployment nginx-deployment

# READY column shows: current/desired
# If stuck, something's wrong
```

**Detailed info:**
```bash
kubectl describe deployment nginx-deployment

# Look at:
# - Replicas section
# - Conditions
# - Events
```

**Check associated ReplicaSet:**
```bash
kubectl get replicaset

# Shows RS created by deployment
kubectl describe replicaset <rs-name>
```

**Check pods:**
```bash
kubectl get pods -l app=nginx

# -l filters by label
# Shows only pods with label app=nginx
```

## The Challenge

Deploy a web application with these requirements:

1. Use image `hashicorp/http-echo:1.0`
2. Run 4 replicas
3. Container listens on port 5678
4. Pass argument `-text=Version 1`
5. Update to Version 2 without downtime
6. Rollback if needed

**Hints:**
```yaml
# Pass args to container:
containers:
- name: app
  image: some-image
  args:
  - "-text=Hello"
  - "-listen=:5678"
```

**Try it yourself.**

<details>
<summary>Solution (click to expand)</summary>

**File: `echo-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Version 1"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
```

**Deploy:**
```bash
kubectl apply -f echo-deployment.yaml
kubectl get pods
# Should see 4 pods

# Update to Version 2
# Edit file, change -text=Version 2
kubectl apply -f echo-deployment.yaml

# Watch rollout
kubectl rollout status deployment echo-deployment

# Rollback if needed
kubectl rollout undo deployment echo-deployment
```

</details>

## Cleanup

```bash
kubectl delete deployment nginx-deployment echo-deployment
```

## Key Takeaways

1. **Use Deployments, not bare Pods** - For any production workload
2. **Self-healing is automatic** - Pods die, Deployment recreates them
3. **Scaling is trivial** - Just change replica count
4. **Rolling updates = zero downtime** - Update apps without service interruption
5. **Rollback is one command** - Recover from bad deploys instantly

## What You Can Do Now

- Deploy apps with multiple replicas
- Scale up/down on demand
- Update apps with rolling updates
- Rollback failed deployments
- Understand deployment strategies

## What's Next

You have multiple pod replicas running. But how do you access them? How does traffic get distributed?

**Next**: [Chapter 4: Services](../04-services/) - Load balancing and service discovery

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Key command**: `kubectl rollout status deployment/<name>` - Watch your updates happen
