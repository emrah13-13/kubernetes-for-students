# Chapter 1: First Contact

Your cluster is running. Now it's time to deploy something and understand what the hell is actually happening.

## The Restaurant Kitchen Analogy

Imagine you own a restaurant. When an order comes in, you don't cook all dishes alone, right? You have:

- **Head Chef** (control plane) - Coordinates all orders, decides who cooks what
- **Stations** (nodes) - Different cooking stations (grill, pasta, dessert)
- **Cooks** (pods) - Actual people executing the work
- **Recipes** (YAML manifests) - Instructions on how to make each dish

Kubernetes is a kitchen management system. You place an order (deploy app), the system decides which station to cook at, assigns a cook, and ensures food comes out on time.

## The Reality: What is a Pod?

**Pod** is the smallest unit you deploy in Kubernetes.

Not a container. Not a deployment. Not a service. **Pod**.

**A Pod is:**
- A wrapper for one or more containers
- Shares network namespace (has same IP address)
- Shares storage volumes
- Scheduled together on the same node
- Mortal - can die anytime and be replaced

**Why not just containers?**  
Because sometimes you need multiple containers that are tightly coupled. Example: main app + logging sidecar. They need to talk via localhost, share files. Pods make this possible.

## Experiment 1: Deploy Your First Pod

Let's start with the simplest thing: run an nginx web server.

**Create file: `first-pod.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Deploy it:**
```bash
kubectl apply -f first-pod.yaml

# Output:
# pod/nginx-pod created
```

**Check status:**
```bash
kubectl get pods

# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          10s
```

**What just happened?**

1. You gave YAML to the Kubernetes API server
2. Scheduler decided which node this pod runs on (in minikube there's only 1 node anyway)
3. Kubelet on that node pulled the nginx image from Docker Hub
4. Container started
5. Pod status became "Running"

## Experiment 2: Interact with Your Pod

**See details:**
```bash
kubectl describe pod nginx-pod
```

You'll see:
- Pod IP address
- Which node it's running on
- Container image details
- Events (pull image, start container, etc)

**Get logs:**
```bash
kubectl logs nginx-pod
```

This is equivalent to `docker logs`. You see output from the container.

**Execute command inside pod:**
```bash
kubectl exec -it nginx-pod -- /bin/bash

# Now you're inside the container
# Try:
curl localhost
# You'll see nginx default page HTML

exit
```

**Access the web server:**

Pods have IPs but they're internal to the cluster. From your laptop you can't access directly. There are 2 ways:

**Option 1: Port forward**
```bash
kubectl port-forward pod/nginx-pod 8080:80

# Open browser: http://localhost:8080
# You'll see "Welcome to nginx!"
```

**Option 2: Exec into pod and curl**
```bash
kubectl exec nginx-pod -- curl localhost
```

## Understanding Pod Lifecycle

Pods are **ephemeral**. Meaning:
- Can die anytime
- If they die, they don't auto-restart (at least not by the pod itself)
- IP address changes every time pod is recreated

**Let's prove it:**

```bash
# Delete the pod
kubectl delete pod nginx-pod

# Try to access it
kubectl get pod nginx-pod
# Error: pod not found

# Check all pods
kubectl get pods
# Empty (or just default k8s system pods)
```

Pod dies, it's gone. No auto-recovery. This is why in production you NEVER deploy bare pods.

## Experiment 3: Multi-Container Pod

Now deploy a pod with 2 containers that share network.

**Create file: `multi-container-pod.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox:1.36
    command: ['sh', '-c', 'while true; do wget -O- localhost; sleep 5; done']
```

**Deploy and observe:**
```bash
kubectl apply -f multi-container-pod.yaml

# Check both containers running
kubectl get pod multi-container-pod

# Output shows 2/2 READY
# NAME                   READY   STATUS    RESTARTS   AGE
# multi-container-pod    2/2     Running   0          15s

# Check logs from specific container
kubectl logs multi-container-pod -c sidecar

# You'll see HTML from nginx, fetched by busybox every 5 seconds
```

**What's happening:**
- Both containers share the same Pod IP
- Sidecar can access nginx via `localhost` because same network namespace
- They're scheduled together, scaled together, die together

## The Breakdown: YAML Structure

Every Kubernetes resource has the same top-level structure:

```yaml
apiVersion: v1           # Which API version
kind: Pod                # What type of resource
metadata:                # Info about the resource
  name: my-pod          # Must be unique in namespace
  labels:               # Key-value pairs for grouping
    app: web
spec:                    # Desired state specification
  containers:           # List of containers
  - name: nginx
    image: nginx:1.25
```

**Key points:**
- `metadata.name` - How you reference this pod later
- `metadata.labels` - Arbitrary tags. Super important for Services (next chapter)
- `spec.containers` - Array, can have multiple
- `image` - Container image. K8s pulls from Docker Hub by default

## Common First-Timer Mistakes

### Mistake 1: Typo in YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    port: 80  # WRONG! Should be "ports"
```

**Result:** Pod won't start, or port not exposed properly.

**Fix:** Always use `ports` (plural) and proper structure.

### Mistake 2: Forgetting Resource Already Exists

```bash
kubectl apply -f first-pod.yaml
# pod/nginx-pod created

kubectl apply -f first-pod.yaml
# pod/nginx-pod unchanged  <- It already exists!
```

If you change the YAML and re-apply, K8s updates it (if possible). But some fields are immutable.

### Mistake 3: Wrong Image Name

```yaml
image: nignx:1.25  # Typo: "nignx" instead of "nginx"
```

**Result:**
```bash
kubectl get pods
# NAME      READY   STATUS         RESTARTS   AGE
# test-pod  0/1     ErrImagePull   0          10s
```

**Check what happened:**
```bash
kubectl describe pod test-pod
# Events will show: Failed to pull image "nignx:1.25": not found
```

## Debugging Pods

**Check pod status:**
```bash
kubectl get pods
```

**Common statuses:**
- `Pending` - Waiting to be scheduled
- `ContainerCreating` - Pulling image, starting container
- `Running` - All containers running
- `Error` or `CrashLoopBackOff` - Container crashed
- `ErrImagePull` - Can't pull image

**See detailed info:**
```bash
kubectl describe pod <pod-name>
```

Look at:
- **Events** section at the bottom - timeline of what happened
- **State** - Current container state
- **Last State** - If container restarted, what was previous state

**See logs:**
```bash
kubectl logs <pod-name>

# If multiple containers:
kubectl logs <pod-name> -c <container-name>

# Follow logs (like tail -f):
kubectl logs -f <pod-name>
```

## The Challenge

Now try it yourself. Goal: Deploy a simple web app that you can access.

**Requirements:**
1. Create pod running `hashicorp/http-echo` image
2. Container should listen on port 5678
3. Set environment variable `TEXT` to "Hello from Kubernetes"
4. Access the app via port-forward
5. Verify it shows your text

**Hints:**
```yaml
# Environment variables in pod spec:
spec:
  containers:
  - name: my-container
    image: some-image
    env:
    - name: ENV_VAR_NAME
      value: "some value"
```

**Command to run the image:**
```bash
# The http-echo image needs args:
args:
- "-text=Hello from Kubernetes"
```

**Try it yourself first.** Solution below.

<details>
<summary>Solution (click to expand)</summary>

**File: `challenge-pod.yaml`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-pod
  labels:
    app: echo
spec:
  containers:
  - name: echo
    image: hashicorp/http-echo:1.0
    args:
    - "-text=Hello from Kubernetes"
    - "-listen=:5678"
    ports:
    - containerPort: 5678
```

**Deploy:**
```bash
kubectl apply -f challenge-pod.yaml
kubectl get pods
kubectl port-forward pod/echo-pod 8080:5678
# Visit http://localhost:8080
```

</details>

## Cleanup

Delete all pods you created:

```bash
kubectl delete pod nginx-pod multi-container-pod echo-pod

# Or delete all pods (careful in real clusters!):
kubectl delete pods --all
```

## Key Takeaways

1. **Pod is the basic unit** - Not containers, not deployments. Pod.
2. **Pods are ephemeral** - They can die anytime. Don't rely on pods staying alive.
3. **kubectl is your interface** - Get, describe, logs, exec - master these commands.
4. **YAML defines desired state** - You declare what you want, K8s makes it happen.
5. **Labels matter** - You'll see why in the next chapter when we talk about Services.

## What You Can Do Now

- Deploy a pod
- Check its status
- Get logs
- Execute commands inside
- Port-forward to access apps
- Understand basic YAML structure

## What You Still Can't Do

- Auto-recover when pod dies
- Scale to multiple instances
- Load balance across pods
- Expose to external traffic properly

That's coming in Chapter 3 (Deployments) and Chapter 4 (Services).

## Next Steps

**Go to**: [Chapter 2: Pods Deep Dive](../02-pods/) - Optional advanced pod concepts  
**Or skip to**: [Chapter 3: Deployments](../03-deployments/) - How to actually run apps in production

---

**Estimated time**: 1-2 hours  
**Difficulty**: Beginner  
**Key command**: `kubectl get pods` - You'll type this thousands of times
