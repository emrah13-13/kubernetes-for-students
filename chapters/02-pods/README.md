# Chapter 2: Pods - Deep Dive

Now you know how to create basic pods. Let's explore advanced pod concepts that you'll use in real applications.

## When Do You Need This Chapter?

**Skip if:**
- You just want to learn deployments and services
- Building simple applications
- Time-constrained learning

**Read if:**
- Need multi-container patterns
- Working with init containers
- Debugging complex pod issues
- Preparing for production deployments
- Curious about pod internals

## Multi-Container Pod Patterns

In Chapter 1, you saw a basic multi-container pod. Here are the standard patterns used in production.

### Pattern 1: Sidecar

**Purpose:** Enhance or extend main container functionality without modifying it.

**Example: Logging Sidecar**

```yaml
# File: sidecar-logging.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # Main application container
  - name: app
    image: busybox:1.36
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date) - App log >> /var/log/app.log; sleep 2; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  
  # Sidecar: Log processor
  - name: log-processor
    image: busybox:1.36
    command: ["/bin/sh"]
    args: ["-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```

**Deploy and test:**
```bash
kubectl apply -f sidecar-logging.yaml

# Main app writes logs
kubectl logs sidecar-demo -c app

# Sidecar reads and processes logs
kubectl logs sidecar-demo -c log-processor
```

**Real-world use cases:**
- Log shipping (Fluentd, Filebeat)
- Metrics collection (Prometheus exporters)
- Configuration synchronization
- TLS termination proxies

### Pattern 2: Adapter

**Purpose:** Standardize and normalize output from main container.

**Example: Monitoring Adapter**

```yaml
# File: adapter-pattern.yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
spec:
  containers:
  # Main app with non-standard metrics format
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
  
  # Adapter: Converts metrics to Prometheus format
  - name: prometheus-adapter
    image: nginx/nginx-prometheus-exporter:0.11
    args:
    - "-nginx.scrape-uri=http://localhost/nginx_status"
    ports:
    - containerPort: 9113
```

**Use cases:**
- Protocol translation
- Data format conversion
- Legacy system integration

### Pattern 3: Ambassador

**Purpose:** Proxy connections to external services.

**Example: Database Ambassador**

```yaml
# File: ambassador-pattern.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-demo
spec:
  containers:
  # Main app connects to localhost
  - name: app
    image: busybox:1.36
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'App running'; sleep 10; done"]
  
  # Ambassador proxies to actual database
  - name: db-proxy
    image: haproxy:2.8
    # In reality, this would proxy to external database
    # Main app connects to localhost, proxy handles routing
```

**Use cases:**
- Database connection pooling
- Service mesh sidecar (Istio, Linkerd)
- Authentication proxy
- Rate limiting

## Init Containers

**Init containers** run before main containers start. Must complete successfully.

**Use cases:**
- Database migrations
- Configuration setup
- Waiting for dependencies
- Pre-warming caches

**Example: Wait for Service**

```yaml
# File: init-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  # Init containers run first, in order
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Waiting for database service..."
      until nslookup mysql-service; do
        echo "DB not ready, waiting..."
        sleep 2
      done
      echo "Database service found!"
  
  - name: setup-data
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Setting up initial data..."
      mkdir -p /work-dir
      echo "Database initialized" > /work-dir/ready
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  
  # Main containers start after init containers succeed
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c']
    args: ['cat /work-dir/ready && sleep 3600']
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  
  volumes:
  - name: workdir
    emptyDir: {}
```

**Deploy and observe:**
```bash
kubectl apply -f init-container.yaml

# Watch init containers complete
kubectl get pod init-demo --watch

# Check init container logs
kubectl logs init-demo -c wait-for-db
kubectl logs init-demo -c setup-data

# Check main container
kubectl logs init-demo -c app
```

**Init container behavior:**
- Run sequentially (one after another)
- Must complete successfully (exit 0)
- If one fails, pod restarts
- Main containers won't start until all init containers succeed

## Pod Lifecycle Phases

Understanding pod states helps with debugging.

**Lifecycle:**
1. **Pending** - Accepted but not scheduled or containers not created
2. **Running** - Pod scheduled, at least one container running
3. **Succeeded** - All containers exited with status 0
4. **Failed** - All containers terminated, at least one failed
5. **Unknown** - Cannot determine state (node communication issue)

**Check pod phase:**
```bash
kubectl get pods
kubectl get pod <name> -o jsonpath='{.status.phase}'
```

**Container states within pod:**
- **Waiting** - Not running yet (pulling image, waiting for init)
- **Running** - Executing
- **Terminated** - Finished or crashed

```bash
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].state}'
```

## Restart Policies

Pods have restart policies that determine behavior when containers exit.

**Three options:**
- `Always` (default) - Always restart container
- `OnFailure` - Restart only if exits with error (non-zero)
- `Never` - Never restart

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-demo
spec:
  restartPolicy: OnFailure  # Only restart if crashes
  containers:
  - name: task
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Task complete" && exit 0']
```

```bash
kubectl apply -f restart-demo.yaml
kubectl get pods --watch

# Pod completes and stays in "Completed" state
# With restartPolicy: Always, it would restart indefinitely
```

**When to use:**
- `Always` - Long-running services (managed by Deployments)
- `OnFailure` - Jobs that should retry on failure
- `Never` - One-time tasks (don't retry even if failed)

## Resource Requests and Limits

Tell Kubernetes how much CPU/memory your container needs.

**Requests vs Limits:**
- **Request** - Minimum guaranteed resources
- **Limit** - Maximum allowed resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"   # Guaranteed minimum
        cpu: "250m"      # 0.25 CPU cores
      limits:
        memory: "128Mi"  # Maximum allowed
        cpu: "500m"      # 0.5 CPU cores
```

**What happens:**
- **Below request:** Pod won't be scheduled (insufficient resources)
- **Between request and limit:** Pod uses what it needs
- **Exceeds memory limit:** Container killed (OOMKilled)
- **Exceeds CPU limit:** Container throttled (slowed down)

**Check resource usage:**
```bash
kubectl top pod resource-demo
```

**Best practices:**
- Always set requests (for scheduling)
- Set limits to prevent resource hogging
- Start conservative, adjust based on monitoring
- Memory: limit = 2x request (rough guideline)
- CPU: limit = 2-4x request

## Liveness and Readiness Probes

Health checks to determine if container is alive and ready.

### Liveness Probe

**Question:** Is the container alive? Should we restart it?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5  # Wait before first check
      periodSeconds: 10       # Check every 10s
      timeoutSeconds: 1       # Probe timeout
      failureThreshold: 3     # Fail after 3 failures
```

**Probe types:**

1. **HTTP GET:**
   ```yaml
   livenessProbe:
     httpGet:
       path: /health
       port: 8080
   ```

2. **TCP Socket:**
   ```yaml
   livenessProbe:
     tcpSocket:
       port: 3306
   ```

3. **Exec Command:**
   ```yaml
   livenessProbe:
     exec:
       command:
       - cat
       - /tmp/healthy
   ```

### Readiness Probe

**Question:** Is the container ready to serve traffic?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
```

**Difference:**
- **Liveness fails:** Container restarted
- **Readiness fails:** Pod removed from Service endpoints (no traffic)

**When pod is not ready:**
```bash
kubectl get pods
# NAME             READY   STATUS    RESTARTS   AGE
# readiness-demo   0/1     Running   0          30s

kubectl get endpoints my-service
# Pod won't be in endpoints until ready
```

### Startup Probe

For slow-starting applications.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
  - name: app
    image: my-slow-app:1.0
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30    # 30 failures allowed
      periodSeconds: 10       # Check every 10s
      # Total: 300s (5 min) to start
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
```

**Use when:**
- App takes long to initialize
- Don't want liveness probe to kill during startup

## Pod Quality of Service (QoS)

Kubernetes assigns QoS classes based on resource configuration.

**Three classes:**

1. **Guaranteed** - Highest priority
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "500m"
     limits:
       memory: "128Mi"  # Same as request
       cpu: "500m"      # Same as request
   ```

2. **Burstable** - Medium priority
   ```yaml
   resources:
     requests:
       memory: "64Mi"
       cpu: "250m"
     limits:
       memory: "128Mi"  # Different from request
       cpu: "500m"
   ```

3. **BestEffort** - Lowest priority (no requests/limits)
   ```yaml
   # No resources specified
   ```

**Why it matters:**
When node runs out of resources, Kubernetes evicts pods in order:
1. BestEffort pods first
2. Burstable pods next
3. Guaranteed pods last

**Check QoS class:**
```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

## Pod Affinity and Anti-Affinity

Control which nodes pods run on, relative to other pods.

### Node Affinity

**Prefer pods run on specific nodes:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx:1.25
```

**Simpler version - nodeSelector:**
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Pod Affinity

**Run pods together on same node:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
  containers:
  - name: app
    image: nginx:1.25
```

### Pod Anti-Affinity

**Spread pods across nodes:**

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname
```

**Use cases:**
- High availability (spread replicas)
- Performance (co-locate cache with app)
- Resource isolation (separate workloads)

## The Challenge

Create a pod with these requirements:

1. **Init container** that waits for a service named "database"
2. **Main container** running nginx
3. **Sidecar container** that tails nginx access logs
4. **Liveness probe** checking nginx on port 80
5. **Resource requests:** 128Mi memory, 250m CPU
6. **Resource limits:** 256Mi memory, 500m CPU

<details>
<summary>Solution (click to expand)</summary>

```yaml
# File: challenge-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: challenge-pod
spec:
  # Init container waits for service
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      until nslookup database; do
        echo "Waiting for database service..."
        sleep 2
      done
  
  # Shared volume for logs
  volumes:
  - name: logs
    emptyDir: {}
  
  containers:
  # Main nginx container
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
    
    # Liveness probe
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    
    # Resources
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
  
  # Sidecar log viewer
  - name: log-viewer
    image: busybox:1.36
    command: ['sh', '-c']
    args: ['tail -f /var/log/nginx/access.log']
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
```

**Deploy (will stay pending until database service exists):**
```bash
kubectl apply -f challenge-pod.yaml

# Create dummy database service to unblock
kubectl create service clusterip database --tcp=3306:3306

# Now pod will start
kubectl get pods --watch
```

</details>

## Cleanup

```bash
kubectl delete pod sidecar-demo adapter-demo ambassador-demo init-demo \
  restart-demo resource-demo liveness-demo readiness-demo \
  startup-demo challenge-pod

kubectl delete service database
```

## Key Takeaways

1. **Multi-container patterns** - Sidecar, Adapter, Ambassador for separation of concerns
2. **Init containers** - Setup tasks before main app starts
3. **Resource management** - Requests for scheduling, limits for protection
4. **Health probes** - Liveness (restart), Readiness (traffic), Startup (slow apps)
5. **QoS classes** - Resource configuration affects eviction priority
6. **Affinity rules** - Control pod placement for HA and performance

## When to Use These Concepts

**In production:**
- Always use resource requests/limits
- Always configure liveness/readiness probes
- Use init containers for dependencies
- Consider QoS for critical workloads

**In development:**
- Can skip probes for quick testing
- Resource limits less critical
- Multi-container patterns as needed

## Next Steps

You now understand pods deeply. Time to manage them at scale.

**Go to**: [Chapter 3: Deployments](../03-deployments/) - Production-ready pod management

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Optional**: Yes, but valuable for production workloads  
**Key concept**: Pods are flexible - use patterns that match your needs
