# Chapter 6: Persistent Storage - Data That Survives

## The Hotel Room Analogy

Imagine you're staying at a hotel. Every time you check out, housekeeping completely resets the room. Your notes, belongings, everything is gone.

Now imagine if the hotel had lockers in the basement where you could store things that survive between stays. That's **persistent storage**.

**In Kubernetes:**
- **Pods are hotel rooms** - When they die, everything inside is gone
- **Persistent Volumes are the lockers** - Storage that survives pod restarts
- **Persistent Volume Claims are your locker key** - Request for storage

## The Reality: Why Do You Need Persistent Storage?

By default, everything in a container is **ephemeral** (temporary).

**What happens when pod restarts:**
```bash
# Create a pod
kubectl run temp-pod --image=busybox:1.36 -- sh -c "echo 'Important data' > /tmp/file.txt && sleep 3600"

# Check the file exists
kubectl exec temp-pod -- cat /tmp/file.txt
# Output: Important data

# Delete and recreate pod
kubectl delete pod temp-pod
kubectl run temp-pod --image=busybox:1.36 -- sh -c "cat /tmp/file.txt 2>/dev/null || echo 'File is gone'"

# Check again
kubectl logs temp-pod
# Output: File is gone
```

**The problem:** Databases, file uploads, logs - all gone when pod restarts.

**The solution:** Persistent storage that exists independently of pod lifecycle.

## Storage Concepts in Kubernetes

There are 3 main storage concepts you need to understand:

### 1. Volumes (Pod-level)
Storage that lives as long as the pod lives. Shared between containers in the pod.

**Types:**
- `emptyDir` - Empty directory created when pod starts
- `hostPath` - Mount directory from node's filesystem
- `configMap` / `secret` - Mount config as files (you saw this in Chapter 5)

### 2. Persistent Volumes (PV)
Cluster-level storage resource. Exists independently of pods.

**Think of it as:** Physical storage available in the cluster.

### 3. Persistent Volume Claims (PVC)
Request for storage by a user. Pods use PVCs to get storage.

**Think of it as:** Storage reservation ticket.

## The Storage Workflow

```
1. Admin creates PV (physical storage)
       ↓
2. User creates PVC (requests storage)
       ↓
3. Kubernetes binds PVC to PV (matches request)
       ↓
4. Pod uses PVC (mounts storage)
```

In **minikube**, PVs are auto-created for you. In real clusters, admins provision them.

## Experiment 1: EmptyDir - Shared Temporary Storage

`emptyDir` is a directory that starts empty when pod is created. All containers in the pod can access it.

**Use case:** Share data between containers in same pod (like logs, cache).

**Create pod with emptyDir:**

```yaml
# File: pod-with-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  # Writer container
  - name: writer
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "$(date): Writer says hello" >> /data/log.txt
        sleep 5
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  # Reader container
  - name: reader
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "=== Latest logs ==="
        tail -5 /data/log.txt 2>/dev/null || echo "No logs yet"
        sleep 10
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

```bash
kubectl apply -f pod-with-emptydir.yaml

# Check writer logs
kubectl logs shared-storage-pod -c writer
# Writes to /data/log.txt

# Check reader logs
kubectl logs shared-storage-pod -c reader
# Reads from /data/log.txt

# Both containers see the same file!
```

**The breakdown:**
- `volumes` section defines the emptyDir volume
- Both containers mount it to `/data`
- Data survives individual container crashes
- Data is DELETED when pod is deleted

**Try it:**
```bash
# Delete and recreate pod
kubectl delete pod shared-storage-pod
kubectl apply -f pod-with-emptydir.yaml

# Check logs
kubectl logs shared-storage-pod -c reader
# Output: No logs yet (data is gone)
```

## Experiment 2: HostPath - Node's Filesystem

`hostPath` mounts a directory from the **node's** filesystem into the pod.

**Warning:** Only works if pod is scheduled on same node. Not portable. Use with caution.

```yaml
# File: pod-with-hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      echo "Writing to node filesystem"
      echo "$(date): Pod was here" >> /host-data/visits.txt
      cat /host-data/visits.txt
      sleep 3600
    volumeMounts:
    - name: host-storage
      mountPath: /host-data
  
  volumes:
  - name: host-storage
    hostPath:
      path: /tmp/k8s-data
      type: DirectoryOrCreate  # Creates dir if not exists
```

```bash
kubectl apply -f pod-with-hostpath.yaml

# Check logs
kubectl logs hostpath-pod
# Shows data written to /tmp/k8s-data on node

# Delete pod
kubectl delete pod hostpath-pod

# Recreate
kubectl apply -f pod-with-hostpath.yaml

# Check logs again
kubectl logs hostpath-pod
# Old data still there! (if scheduled on same node)
```

**When to use hostPath:**
- Accessing node's Docker socket (`/var/run/docker.sock`)
- Reading node logs
- Development/testing on single-node clusters

**When NOT to use:**
- Production apps (not portable)
- Multi-node clusters (data is node-specific)

## Experiment 3: Persistent Volume Claim - Real Persistence

Now the real stuff. PVCs are how you get storage that truly survives.

### Step 1: Create PVC

```yaml
# File: pvc-demo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce  # Can be mounted by single node
  resources:
    requests:
      storage: 1Gi  # Request 1GB storage
  storageClassName: standard  # Storage class (auto in minikube)
```

```bash
kubectl apply -f pvc-demo.yaml

# Check PVC status
kubectl get pvc
# NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES
# my-pvc   Bound    pvc-abc123...                             1Gi        RWO

# Check PV (auto-created in minikube)
kubectl get pv
# Shows the PV that was automatically provisioned
```

**Access Modes explained:**
- `ReadWriteOnce` (RWO) - Mount by single node (most common)
- `ReadOnlyMany` (ROX) - Mount by multiple nodes, read-only
- `ReadWriteMany` (RWX) - Mount by multiple nodes, read-write

### Step 2: Use PVC in Pod

```yaml
# File: pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c']
    args:
    - |
      # Check if data file exists
      if [ -f /data/counter.txt ]; then
        COUNTER=$(cat /data/counter.txt)
        echo "Previous counter value: $COUNTER"
        COUNTER=$((COUNTER + 1))
      else
        echo "First run, initializing counter"
        COUNTER=1
      fi
      
      echo $COUNTER > /data/counter.txt
      echo "Current counter: $COUNTER"
      echo "Sleeping for 1 hour..."
      sleep 3600
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
kubectl apply -f pod-with-pvc.yaml

# Check logs
kubectl logs pvc-pod
# Output: First run, initializing counter
#         Current counter: 1

# Delete pod
kubectl delete pod pvc-pod

# Recreate pod
kubectl apply -f pod-with-pvc.yaml

# Check logs again
kubectl logs pvc-pod
# Output: Previous counter value: 1
#         Current counter: 2

# Data survived! PVC still exists.
```

**The magic:** PVC exists independently. Pod can be deleted and recreated, data remains.

### Step 3: Verify Persistence

```bash
# Delete pod multiple times
kubectl delete pod pvc-pod
kubectl apply -f pod-with-pvc.yaml
kubectl logs pvc-pod
# Counter: 3

kubectl delete pod pvc-pod
kubectl apply -f pod-with-pvc.yaml
kubectl logs pvc-pod
# Counter: 4

# PVC keeps data across pod restarts!
```

## Experiment 4: Database with Persistent Storage

Real-world example: MySQL database with persistent data.

```yaml
# File: mysql-with-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password123"
    - name: MYSQL_DATABASE
      value: "testdb"
    ports:
    - containerPort: 3306
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql  # MySQL data directory
  
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

```bash
kubectl apply -f mysql-with-storage.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod/mysql --timeout=60s

# Create some data
kubectl exec -it mysql -- mysql -uroot -ppassword123 testdb -e "
  CREATE TABLE users (id INT, name VARCHAR(50));
  INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');
  SELECT * FROM users;
"
# Output:
# +------+-------+
# | id   | name  |
# +------+-------+
# |    1 | Alice |
# |    2 | Bob   |
# +------+-------+

# Delete pod
kubectl delete pod mysql

# Recreate
kubectl apply -f mysql-with-storage.yaml
kubectl wait --for=condition=ready pod/mysql --timeout=60s

# Check data still exists
kubectl exec -it mysql -- mysql -uroot -ppassword123 testdb -e "SELECT * FROM users;"
# Data is still there!
```

## Storage Classes

**StorageClass** defines different "types" of storage with different characteristics.

```bash
# Check available storage classes
kubectl get storageclass

# In minikube:
# NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate
```

**What they control:**
- **Provisioner** - What creates the storage (AWS EBS, GCP Disk, local, etc.)
- **Reclaim Policy** - What happens to PV when PVC is deleted
  - `Delete` - PV is deleted (data gone)
  - `Retain` - PV kept for manual cleanup
- **Volume Binding Mode** - When PV is created
  - `Immediate` - Created when PVC is created
  - `WaitForFirstConsumer` - Created when pod uses it

**Create PVC with specific storage class:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd  # Specific storage class
  resources:
    requests:
      storage: 5Gi
```

## Experiment 5: StatefulSet with Persistent Storage

For databases and stateful apps, you use **StatefulSet** instead of Deployment.

**StatefulSet provides:**
- Stable pod names (pod-0, pod-1, pod-2)
- Ordered deployment and scaling
- Automatic PVC creation per pod

```yaml
# File: statefulset-demo.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None  # Headless service
  selector:
    app: nginx-stateful
  ports:
  - port: 80

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        command: ['sh', '-c']
        args:
        - |
          # Write pod name to index.html
          echo "<h1>Hello from $HOSTNAME</h1>" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
  
  # VolumeClaimTemplate creates PVC per pod
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f statefulset-demo.yaml

# Watch pods being created in order
kubectl get pods -w
# web-0   0/1   Creating
# web-0   1/1   Running
# web-1   0/1   Creating  # Only starts after web-0 is ready
# web-1   1/1   Running
# web-2   0/1   Creating
# web-2   1/1   Running

# Check PVCs (one per pod)
kubectl get pvc
# www-web-0   Bound   pvc-abc...   1Gi
# www-web-1   Bound   pvc-def...   1Gi
# www-web-2   Bound   pvc-ghi...   1Gi

# Each pod has its own storage
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# <h1>Hello from web-0</h1>

kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
# <h1>Hello from web-1</h1>

# Delete a pod
kubectl delete pod web-1

# StatefulSet recreates it with SAME NAME
kubectl get pods
# web-1 gets recreated (not web-1-xyz like Deployment)

# And uses the SAME PVC
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
# <h1>Hello from web-1</h1>  # Same data!
```

## The Breakdown: PV vs PVC vs StorageClass

```
┌─────────────────────┐
│   StorageClass      │  "Type" of storage (SSD, HDD, cloud provider)
│   (Admin creates)   │  Automatically provisions PVs
└──────────┬──────────┘
           │
           │ provisions
           ↓
┌─────────────────────┐
│ PersistentVolume    │  Actual storage resource in cluster
│ (Auto-created)      │  Has size, access mode, storage backend
└──────────┬──────────┘
           │
           │ binds to
           ↓
┌─────────────────────┐
│ PVC                 │  User's request for storage
│ (User creates)      │  "I need 5GB of storage"
└──────────┬──────────┘
           │
           │ used by
           ↓
┌─────────────────────┐
│ Pod                 │  Consumes the storage
│ (User creates)      │  Mounts PVC as volume
└─────────────────────┘
```

## Common Storage Patterns

### Pattern 1: Shared Read-Only Config

```yaml
# ConfigMap mounted as volume (Chapter 5)
volumes:
- name: config
  configMap:
    name: app-config
```

**Use for:** Configuration files, static content

### Pattern 2: Temporary Scratch Space

```yaml
# emptyDir for temporary files
volumes:
- name: cache
  emptyDir: {}
```

**Use for:** Cache, temporary processing files

### Pattern 3: Persistent Database Storage

```yaml
# PVC for databases
volumes:
- name: db-storage
  persistentVolumeClaim:
    claimName: postgres-pvc
```

**Use for:** Databases, file uploads, any data that must survive

### Pattern 4: Shared Storage Across Pods

```yaml
# PVC with ReadWriteMany (if supported)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteMany  # Multiple pods can use it
  resources:
    requests:
      storage: 10Gi
```

**Use for:** Shared file systems, media processing pipelines

## The Challenge

Deploy a Redis instance with persistent storage that survives pod restarts.

**Requirements:**
1. Create PVC requesting 1Gi storage
2. Deploy Redis pod using the PVC
3. Store some data in Redis
4. Delete and recreate the pod
5. Verify data is still there

**Hints:**

```yaml
# Redis stores data in /data
volumeMounts:
- name: redis-storage
  mountPath: /data

# Redis image
image: redis:7.2

# To test Redis:
kubectl exec -it <pod> -- redis-cli
> SET mykey "Hello"
> GET mykey
```

**Try it yourself first.** Solution below.

<details>
<summary>Solution (click to expand)</summary>

**File: `redis-persistent.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: redis
  labels:
    app: redis
spec:
  containers:
  - name: redis
    image: redis:7.2
    ports:
    - containerPort: 6379
    volumeMounts:
    - name: redis-storage
      mountPath: /data
    command: ['redis-server', '--appendonly', 'yes']
  
  volumes:
  - name: redis-storage
    persistentVolumeClaim:
      claimName: redis-pvc
```

**Test it:**
```bash
# Deploy
kubectl apply -f redis-persistent.yaml

# Wait for ready
kubectl wait --for=condition=ready pod/redis --timeout=60s

# Store data
kubectl exec -it redis -- redis-cli SET name "Kubernetes Student"
kubectl exec -it redis -- redis-cli SET course "K8s Storage"

# Verify
kubectl exec -it redis -- redis-cli GET name
# Output: "Kubernetes Student"

# Delete pod
kubectl delete pod redis

# Recreate
kubectl apply -f redis-persistent.yaml
kubectl wait --for=condition=ready pod/redis --timeout=60s

# Check data still exists
kubectl exec -it redis -- redis-cli GET name
# Output: "Kubernetes Student"

kubectl exec -it redis -- redis-cli GET course
# Output: "K8s Storage"

# Data survived!
```

**Explanation:**
- `--appendonly yes` enables Redis persistence
- Data is written to `/data` directory
- PVC ensures `/data` survives pod restarts
- Each pod restart reconnects to same PVC

</details>

## Common Mistakes

### Mistake 1: Forgetting PVC Persists After Pod Deletion

```bash
# Delete pod
kubectl delete pod my-pod

# PVC still exists!
kubectl get pvc
# my-pvc still there

# To fully clean up:
kubectl delete pvc my-pvc
```

**Why it matters:** PVCs cost money in cloud environments even if unused.

### Mistake 2: Using ReadWriteMany Without Support

```yaml
accessModes:
- ReadWriteMany  # Not all storage classes support this!
```

**Error:**
```
PersistentVolumeClaim is not bound: no persistent volumes available
```

**Solution:** Check storage class capabilities. Most only support ReadWriteOnce.

### Mistake 3: Deleting PVC While Pod Uses It

```bash
kubectl delete pvc my-pvc
# Hangs... PVC stuck in "Terminating"
```

**Why:** PVC is protected while in use.

**Solution:** Delete pod first, then PVC.

### Mistake 4: Not Setting Resource Limits

```yaml
# Pod can fill entire disk
volumeMounts:
- name: logs
  mountPath: /var/log
```

**Solution:** Use ephemeral storage limits:

```yaml
resources:
  limits:
    ephemeral-storage: "2Gi"
```

### Mistake 5: Expecting Data Across Different PVCs

```bash
# Create pod with PVC
kubectl apply -f pod-with-pvc-A.yaml

# Delete pod and PVC
kubectl delete pod my-pod
kubectl delete pvc pvc-A

# Create new PVC with different name
kubectl apply -f pod-with-pvc-B.yaml

# Data is gone! (different PVC = different storage)
```

**Solution:** Keep same PVC name if you want same data.

## Cleanup

```bash
kubectl delete pod shared-storage-pod hostpath-pod pvc-pod mysql redis
kubectl delete statefulset web
kubectl delete service nginx-headless
kubectl delete pvc my-pvc mysql-pvc redis-pvc
kubectl delete pvc www-web-0 www-web-1 www-web-2  # StatefulSet PVCs
```

## Key Takeaways

1. **Pods are ephemeral** - By default, data is lost when pod restarts
2. **emptyDir for sharing** - Temporary storage shared between containers in pod
3. **hostPath for node access** - Use cautiously, not portable
4. **PVC for real persistence** - Storage that survives pod lifecycle
5. **StatefulSet for stateful apps** - Stable names + automatic PVC per pod
6. **Access modes matter** - Most storage only supports ReadWriteOnce

## When to Use What

| Storage Type | Use When | Survives Pod Restart? |
|-------------|----------|----------------------|
| emptyDir | Temporary cache, scratch space | No |
| hostPath | Node logs, Docker socket | Yes (if same node) |
| configMap | Static config files | N/A (not for data) |
| secret | Credentials, certs | N/A (not for data) |
| PVC (RWO) | Database, single-pod app data | Yes |
| PVC (RWX) | Shared files, media processing | Yes |
| StatefulSet + PVC | Databases, distributed systems | Yes |

## What You Can Do Now

- Create persistent storage that survives pod restarts
- Use PVCs for databases and stateful applications
- Share data between containers with emptyDir
- Deploy StatefulSets for stable pod identities
- Understand storage classes and access modes
- Choose right storage pattern for your use case

## What's Next

You can store data. But how do you expose apps to the internet? NodePort isn't production-ready.

**Next**: Chapter 7: Ingress - The Front Door (Coming Soon)

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Key command**: `kubectl get pvc` - Check your persistent storage claims
