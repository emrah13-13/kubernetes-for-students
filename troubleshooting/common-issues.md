# Common Issues and Solutions

When learning Kubernetes, you'll hit these issues. Here's how to fix them.

## Pod Issues

### Pod Stuck in "Pending"

**Symptom:**
```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# my-pod  0/1     Pending   0          5m
```

**Causes & Solutions:**

1. **Insufficient resources**
   ```bash
   kubectl describe pod my-pod
   # Look for: "Insufficient memory" or "Insufficient cpu"
   ```
   **Fix:** Reduce resource requests or add more nodes
   ```yaml
   resources:
     requests:
       memory: "64Mi"  # Lower this
       cpu: "100m"
   ```

2. **No nodes available**
   ```bash
   kubectl get nodes
   # Check if nodes are Ready
   ```
   **Fix:** Start minikube or check node health

3. **PersistentVolumeClaim not bound**
   ```bash
   kubectl describe pod my-pod
   # Look for: "waiting for volume to be created"
   ```
   **Fix:** Check PVC status, may need to create PV

### Pod in "ImagePullBackOff" or "ErrImagePull"

**Symptom:**
```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# my-pod  0/1     ImagePullBackOff   0          2m
```

**Causes & Solutions:**

1. **Wrong image name/tag**
   ```bash
   kubectl describe pod my-pod
   # Look for: "Failed to pull image"
   ```
   **Fix:** Correct the image name
   ```yaml
   image: nginx:1.25  # Check spelling and tag
   ```

2. **Private registry without credentials**
   ```bash
   kubectl describe pod my-pod
   # Look for: "unauthorized" or "authentication required"
   ```
   **Fix:** Add imagePullSecrets (covered in Chapter 5)

3. **Network issues**
   **Fix:** Check internet connection, docker hub status

### Pod in "CrashLoopBackOff"

**Symptom:**
```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# my-pod  0/1     CrashLoopBackOff   5          3m
```

**Diagnosis:**
```bash
# Check logs
kubectl logs my-pod

# Check previous container logs (if restarted)
kubectl logs my-pod --previous

# Describe for events
kubectl describe pod my-pod
```

**Common Causes:**

1. **Application crashes immediately**
   - Fix: Check logs for error messages
   - Application might need config/environment variables

2. **Wrong command or args**
   ```yaml
   command: ["/bin/sh"]
   args: ["-c", "wrong-command"]  # This doesn't exist
   ```
   **Fix:** Verify command/args are correct

3. **Missing dependencies**
   - Application needs database but can't connect
   - Fix: Check service names, ports, credentials

### Pod Ready but Not Serving Traffic

**Diagnosis:**
```bash
# Check if container is listening on correct port
kubectl exec my-pod -- netstat -tulpn
kubectl exec my-pod -- ss -tulpn  # alternative

# Test from within cluster
kubectl run test --image=busybox --rm -it -- wget -O- http://pod-ip:port
```

**Fixes:**
- Verify containerPort matches app's listening port
- Check firewall/security rules
- Verify health checks not failing

## Deployment Issues

### Deployment Not Creating Pods

**Check:**
```bash
kubectl get deployment
# READY column shows 0/3

kubectl describe deployment my-deployment
# Look at Events section
```

**Common Causes:**

1. **Selector doesn't match template labels**
   ```yaml
   spec:
     selector:
       matchLabels:
         app: web  # Doesn't match
     template:
       metadata:
         labels:
           app: nginx  # Mismatch!
   ```
   **Fix:** Make sure they match exactly

2. **Invalid YAML**
   ```bash
   kubectl apply -f deployment.yaml
   # Error: error validating data
   ```
   **Fix:** Check YAML syntax, indentation

### Rollout Stuck

**Symptom:**
```bash
kubectl rollout status deployment/my-deploy
# Waiting for deployment "my-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
```

**Check:**
```bash
kubectl describe deployment my-deploy
kubectl get pods -l app=my-app
```

**Common Causes:**
- New image has issues (CrashLoopBackOff)
- Insufficient resources for new pods
- ReadinessProbe failing

**Fix:**
```bash
# Rollback if needed
kubectl rollout undo deployment/my-deploy
```

## Service Issues

### Service Has No Endpoints

**Symptom:**
```bash
kubectl get endpoints my-service
# NAME         ENDPOINTS   AGE
# my-service   <none>      5m
```

**Diagnosis:**
```bash
kubectl describe service my-service
# Check Selector

kubectl get pods --show-labels
# Check if any pods have matching labels
```

**Causes & Fixes:**

1. **Selector doesn't match pod labels**
   ```yaml
   # Service
   selector:
     app: web
   
   # Pods
   labels:
     app: nginx  # Mismatch!
   ```
   **Fix:** Match the selectors

2. **Pods not ready**
   ```bash
   kubectl get pods
   # Pods might be Pending or CrashLoopBackOff
   ```
   **Fix:** Fix pod issues first

3. **No pods exist**
   - Create deployment first

### Can't Access Service

**From inside cluster:**
```bash
# Create test pod
kubectl run test --image=busybox --rm -it -- sh

# Try to reach service
wget -O- http://service-name:port
nslookup service-name
```

**From outside cluster (NodePort):**
```bash
# Get node IP
minikube ip

# Get NodePort
kubectl get svc my-service
# Look at PORT(S) column: 80:30080/TCP
#                              ^^^^^
#                              NodePort

# Try accessing
curl http://$(minikube ip):30080
```

**Common Issues:**
- Wrong port (use port, not targetPort for ClusterIP)
- Wrong service name or namespace
- Firewall blocking NodePort
- Service type is ClusterIP (can't access from outside)

## Minikube Issues

### Minikube Won't Start

**Error: "Insufficient memory"**
```bash
minikube start --memory=4096 --cpus=2
```

**Error: "Driver not found"**
```bash
# List available drivers
minikube start --help | grep driver

# Specify driver
minikube start --driver=docker
```

**Error: "Another hypervisor is running"**
- Disable Hyper-V or VirtualBox
- Or use docker driver

**Complete reset:**
```bash
minikube delete
minikube start
```

### Minikube Cluster Unreachable

```bash
# Check status
minikube status

# Restart
minikube stop
minikube start

# Check logs
minikube logs
```

## YAML Syntax Issues

### Invalid Indentation

**Error:**
```
error: error parsing deployment.yaml: error converting YAML to JSON
```

**Fix:**
- YAML uses 2 spaces for indentation
- Never use tabs
- Check alignment

**Valid:**
```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
```

**Invalid:**
```yaml
spec:
replicas: 3  # Wrong indentation
  selector:
    matchLabels:
  app: nginx  # Wrong indentation
```

### Missing Required Fields

**Error:**
```
error: error validating "file.yaml": error validating data: 
ValidationError(Deployment.spec): missing required field "selector"
```

**Fix:** Add the missing field
```yaml
spec:
  selector:  # Required!
    matchLabels:
      app: nginx
```

## Permission Issues

### "Forbidden" Errors

**Error:**
```bash
Error from server (Forbidden): pods is forbidden: 
User "system:serviceaccount:default:default" cannot list pods in namespace "default"
```

**Causes:**
- RBAC restrictions
- Wrong kubeconfig context

**Check:**
```bash
kubectl auth can-i list pods
kubectl auth can-i create deployments
```

## DNS Issues

### Service Discovery Not Working

**Test DNS:**
```bash
kubectl run test --image=busybox --rm -it -- sh

# Inside pod
nslookup kubernetes
# Should resolve

nslookup my-service
# Should resolve to service IP
```

**If DNS doesn't work:**
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

## Resource Quota Issues

**Error:**
```
Error creating: pods "my-pod" is forbidden: 
exceeded quota: compute-resources
```

**Check quotas:**
```bash
kubectl describe resourcequota
```

**Fix:** Delete unused resources or increase quota

## Debugging Workflow

When something doesn't work, follow this:

1. **Check pod status**
   ```bash
   kubectl get pods
   ```

2. **Describe the resource**
   ```bash
   kubectl describe pod/deployment/service <name>
   # Read Events section carefully
   ```

3. **Check logs**
   ```bash
   kubectl logs <pod-name>
   kubectl logs <pod-name> --previous  # if restarted
   ```

4. **Check service/endpoints**
   ```bash
   kubectl get endpoints <service-name>
   ```

5. **Test connectivity**
   ```bash
   kubectl run test --image=busybox --rm -it -- sh
   # Then wget/curl from inside
   ```

6. **Check events**
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   ```

## Quick Fixes Checklist

- [ ] Is minikube running? `minikube status`
- [ ] Are nodes ready? `kubectl get nodes`
- [ ] Check pod status: `kubectl get pods`
- [ ] Check logs: `kubectl logs <pod>`
- [ ] Describe resource: `kubectl describe <type> <name>`
- [ ] Check selectors match: Service selector = Pod labels
- [ ] Check ports: containerPort = targetPort
- [ ] Image name correct? No typos?
- [ ] YAML valid? Proper indentation?
- [ ] Enough resources? `kubectl top nodes`

## Still Stuck?

1. **Delete and recreate:**
   ```bash
   kubectl delete -f file.yaml
   kubectl apply -f file.yaml
   ```

2. **Check official docs:** https://kubernetes.io/docs/

3. **Search error message:** Copy exact error to Google

4. **Ask for help:** 
   - GitHub issues on this repo
   - Kubernetes Slack
   - Stack Overflow with tag `kubernetes`

Remember: Error messages are your friends. Read them carefully.
