# Chapter 4: Services - The Traffic Director

You have multiple pod replicas running. But pod IPs change when they restart. How do you access them reliably? Enter Services.

## The Hotel Receptionist Analogy

Imagine a hotel where:
- Guests (pods) check in and out constantly
- Room numbers (IP addresses) change for privacy
- You need to meet your friend staying there

You don't track which room they're in. You ask the **receptionist** (Service):
- "I need to reach anyone from the Johnson party" (label selector)
- Receptionist knows current room numbers
- Routes you to the right person
- If someone's not available, routes to another family member

**Services** are Kubernetes' receptionists. They provide stable endpoints to access dynamic pods.

## The Reality: What is a Service?

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them.

**Problems Services solve:**
- Pod IPs change when pods restart
- Need to load balance across multiple pods
- Want a single endpoint for a group of pods
- Don't want to track individual pod IPs

**Service provides:**
- **Stable IP and DNS name** - Won't change even if pods do
- **Load balancing** - Distributes traffic across healthy pods
- **Service discovery** - Other apps can find your service by name

## Service Types

There are 4 types of Services:

1. **ClusterIP** (default) - Internal cluster access only
2. **NodePort** - Expose on each node's IP at a static port
3. **LoadBalancer** - Cloud provider's load balancer
4. **ExternalName** - Maps to external DNS name

We'll focus on ClusterIP and NodePort for learning.

## Experiment 1: ClusterIP Service

This is the most common type - for internal communication between services.

**First, create a deployment:**

```yaml
# File: nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx  # Service will select pods by this label
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide
# Note the different IPs for each pod
```

**Now create a Service:**

```yaml
# File: nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP  # Default, can omit
  selector:
    app: nginx  # Select pods with label app=nginx
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 80  # Pod port
```

```bash
kubectl apply -f nginx-service.yaml

kubectl get service nginx-service
# Output:
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.100.123   <none>        80/TCP    10s
```

**Test the service:**

```bash
# Create a test pod to curl from
kubectl run test-pod --image=busybox:1.36 --rm -it --restart=Never -- sh

# Inside the test pod:
wget -qO- nginx-service
# You'll see nginx HTML

# Try multiple times
for i in $(seq 1 10); do wget -qO- nginx-service | grep title; done

# Notice requests go to different pods (if they logged, you'd see different pod IPs handling requests)

exit
```

**What happened?**
- Service has a stable IP (ClusterIP)
- DNS entry created: `nginx-service` resolves to that IP
- Service load balances across all pods matching selector
- Works even if pods die and are recreated with new IPs

## Understanding Service Discovery

In Kubernetes, you can reach services by name:

```bash
# From any pod in the same namespace:
curl nginx-service

# From a different namespace:
curl nginx-service.default.svc.cluster.local
```

Format: `<service-name>.<namespace>.svc.cluster.local`

This is **DNS-based service discovery**. No need to hardcode IPs.

## Experiment 2: NodePort Service

NodePort exposes the service on each node's IP at a static port. Useful for development or when you don't have a LoadBalancer.

**Create NodePort service:**

```yaml
# File: nginx-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80        # Service port (ClusterIP)
    targetPort: 80  # Pod port
    nodePort: 30080 # Port on nodes (30000-32767 range)
```

```bash
kubectl apply -f nginx-nodeport.yaml

kubectl get service nginx-nodeport
# Output:
# NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# nginx-nodeport   NodePort   10.96.200.50    <none>        80:30080/TCP   10s
```

**Access it:**

```bash
# Get minikube IP
minikube ip
# Example: 192.168.49.2

# Access via browser or curl
curl http://192.168.49.2:30080

# Or use minikube service command
minikube service nginx-nodeport
# This opens browser automatically
```

**What's happening?**
- Service accessible at `<NodeIP>:30080`
- In minikube, only one node, so one IP
- In multi-node cluster, works on ANY node's IP
- Traffic forwarded to ClusterIP, then load balanced to pods

## Experiment 3: Service Load Balancing

Let's prove load balancing actually works.

**Deploy app that shows pod name:**

```yaml
# File: whoami-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: containous/whoami:latest
        ports:
        - containerPort: 80
```

**Create service:**

```yaml
# File: whoami-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami-service
spec:
  type: NodePort
  selector:
    app: whoami
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30081
```

```bash
kubectl apply -f whoami-deployment.yaml
kubectl apply -f whoami-service.yaml

# Access multiple times
for i in {1..10}; do curl -s http://$(minikube ip):30081 | grep Hostname; done
```

You'll see different pod names, proving load balancing.

## Experiment 4: Endpoints

Services don't directly know about pods. They use **Endpoints**.

```bash
kubectl get endpoints nginx-service

# Output shows IPs of pods backing the service:
# NAME            ENDPOINTS                                   AGE
# nginx-service   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80   5m
```

When pods die/created, endpoints automatically update.

**Watch it change:**

```bash
# Terminal 1: Watch endpoints
kubectl get endpoints nginx-service --watch

# Terminal 2: Kill a pod
kubectl delete pod <pod-name>

# See endpoints update in real-time
```

## The Breakdown: Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service

spec:
  type: ClusterIP  # or NodePort, LoadBalancer

  selector:
    app: nginx     # Pods with this label are part of service

  ports:
  - protocol: TCP
    port: 80       # Service listens on this port
    targetPort: 80 # Forward to this port on pods
    # nodePort: 30080  # Only for NodePort type
```

**Key points:**
- `selector` must match pod labels
- `port` is what clients connect to
- `targetPort` is where traffic goes on pods (can be port number or name)
- Multiple ports supported (just list multiple items)

## Common Mistakes

### Mistake 1: Selector Mismatch

```yaml
# Service
selector:
  app: web

# Pods labeled:
labels:
  app: nginx
```

**Result:** Service has no endpoints.

```bash
kubectl get endpoints my-service
# ENDPOINTS: <none>
```

**Fix:** Match the labels exactly.

### Mistake 2: Wrong targetPort

```yaml
# Service
ports:
- port: 80
  targetPort: 8080  # Wrong! Container listens on 80

# Container
ports:
- containerPort: 80
```

**Result:** Connection refused errors.

**Fix:** `targetPort` must match container's listening port.

### Mistake 3: NodePort Range

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 8080  # ERROR! Must be 30000-32767
```

**Result:** Error creating service.

**Fix:** Use port in valid range or omit (K8s assigns automatically).

## Session Affinity

By default, service load balances randomly. Sometimes you want sticky sessions (same client goes to same pod).

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour
```

Now same client IP always routes to same pod (for timeout duration).

## Headless Services

Sometimes you don't want load balancing. You want to discover all pod IPs directly.

```yaml
spec:
  clusterIP: None  # Headless service
  selector:
    app: nginx
```

**Use cases:**
- Stateful applications
- Custom load balancing logic
- Direct pod-to-pod communication

```bash
# DNS returns all pod IPs, not service IP
nslookup nginx-service
```

## The Challenge

Create a multi-tier application:

1. **Backend deployment**:
   - Image: `hashicorp/http-echo:1.0`
   - 3 replicas
   - Args: `-text=Backend Response`
   - Listens on port 5678

2. **Backend service**:
   - ClusterIP type
   - Named `backend-service`

3. **Frontend pod**:
   - Image: `busybox:1.36`
   - Should be able to curl `backend-service:5678`

4. **Verify**:
   - Frontend can reach backend by service name
   - Load balancing works across 3 backend pods

**Try it yourself.**

<details>
<summary>Solution (click to expand)</summary>

**backend-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Backend Response"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
```

**backend-service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 5678
    targetPort: 5678
```

**Deploy and test:**
```bash
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# Test from frontend
kubectl run frontend --image=busybox:1.36 --rm -it --restart=Never -- sh
wget -qO- backend-service:5678
# Should show: Backend Response
```

</details>

## Cleanup

```bash
kubectl delete deployment nginx-deployment whoami backend
kubectl delete service nginx-service nginx-nodeport whoami-service backend-service
```

## Key Takeaways

1. **Services provide stable endpoints** - Pod IPs change, Service IP doesn't
2. **DNS-based discovery** - Access services by name, not IP
3. **Automatic load balancing** - Traffic distributed across healthy pods
4. **Label selectors connect them** - Service finds pods via labels
5. **Types for different needs** - ClusterIP for internal, NodePort for external access

## What You Can Do Now

- Create services for deployments
- Use service discovery via DNS
- Expose services externally with NodePort
- Understand load balancing
- Debug service connectivity issues

## What's Next

You can deploy apps and expose them. But configuration is hardcoded. What if you need different config for dev/staging/prod?

**Next**: [Chapter 5: ConfigMaps & Secrets](../05-config/) - Externalize configuration

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Key command**: `kubectl get endpoints` - Check what pods back your service
