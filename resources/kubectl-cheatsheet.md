# kubectl Cheatsheet

Quick reference for the most common kubectl commands you'll use.

## Cluster Info

```bash
# Check cluster status
kubectl cluster-info

# View cluster nodes
kubectl get nodes

# Detailed node info
kubectl describe node <node-name>

# Check kubectl version
kubectl version --client
```

## Working with Pods

```bash
# List all pods
kubectl get pods

# List pods with more details
kubectl get pods -o wide

# List pods in all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # shorthand

# Watch pods in real-time
kubectl get pods --watch

# Describe a pod (detailed info)
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>

# Follow logs (like tail -f)
kubectl logs -f <pod-name>

# Logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Execute command in pod
kubectl exec <pod-name> -- <command>

# Interactive shell in pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- sh  # for alpine/busybox

# Port forward to access pod locally
kubectl port-forward pod/<pod-name> <local-port>:<pod-port>
# Example: kubectl port-forward pod/nginx 8080:80

# Delete a pod
kubectl delete pod <pod-name>

# Delete all pods
kubectl delete pods --all
```

## Working with Deployments

```bash
# List deployments
kubectl get deployments
kubectl get deploy  # shorthand

# Create deployment from YAML
kubectl apply -f deployment.yaml

# Create deployment imperatively
kubectl create deployment nginx --image=nginx:1.25

# Scale deployment
kubectl scale deployment <name> --replicas=5

# Update deployment image
kubectl set image deployment/<name> <container>=<new-image>
# Example: kubectl set image deployment/nginx nginx=nginx:1.26

# Check rollout status
kubectl rollout status deployment/<name>

# View rollout history
kubectl rollout history deployment/<name>

# Rollback to previous version
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=2

# Pause rollout
kubectl rollout pause deployment/<name>

# Resume rollout
kubectl rollout resume deployment/<name>

# Restart deployment (recreate all pods)
kubectl rollout restart deployment/<name>

# Delete deployment
kubectl delete deployment <name>
```

## Working with Services

```bash
# List services
kubectl get services
kubectl get svc  # shorthand

# Create service from YAML
kubectl apply -f service.yaml

# Expose deployment as service
kubectl expose deployment <name> --port=80 --target-port=8080

# Create NodePort service
kubectl expose deployment <name> --type=NodePort --port=80

# Get service details
kubectl describe service <name>

# Get service endpoints
kubectl get endpoints <name>

# Delete service
kubectl delete service <name>

# Minikube: Access NodePort service
minikube service <service-name>
```

## Working with YAML Files

```bash
# Apply YAML file
kubectl apply -f file.yaml

# Apply all YAML in directory
kubectl apply -f ./directory/

# Apply from URL
kubectl apply -f https://example.com/resource.yaml

# Delete resources from YAML
kubectl delete -f file.yaml

# View YAML of running resource
kubectl get pod <name> -o yaml
kubectl get deployment <name> -o yaml

# Dry run (validate without creating)
kubectl apply -f file.yaml --dry-run=client

# Server-side dry run
kubectl apply -f file.yaml --dry-run=server
```

## Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns  # shorthand

# Create namespace
kubectl create namespace <name>

# Set default namespace for commands
kubectl config set-context --current --namespace=<name>

# Get resources from specific namespace
kubectl get pods -n <namespace>

# Get resources from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A
```

## Labels and Selectors

```bash
# Show labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=nginx
kubectl get pods --selector=app=nginx

# Multiple labels (AND)
kubectl get pods -l app=nginx,env=prod

# Multiple labels (OR)
kubectl get pods -l 'app in (nginx, apache)'

# Add label to resource
kubectl label pod <name> env=prod

# Remove label
kubectl label pod <name> env-

# Update label
kubectl label pod <name> env=staging --overwrite
```

## Resource Management

```bash
# View resource usage
kubectl top nodes
kubectl top pods

# Set resource limits
kubectl set resources deployment <name> \
  --limits=cpu=500m,memory=256Mi \
  --requests=cpu=250m,memory=128Mi
```

## Debugging

```bash
# Describe resource (shows events)
kubectl describe <resource-type> <name>

# Get events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'

# Check pod status reasons
kubectl get pods
kubectl describe pod <name> | grep -A 10 Events

# Run temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Copy files from/to pod
kubectl cp <pod>:/path/to/file ./local-file
kubectl cp ./local-file <pod>:/path/to/file
```

## Output Formats

```bash
# JSON output
kubectl get pods -o json

# YAML output
kubectl get pods -o yaml

# Wide output (more columns)
kubectl get pods -o wide

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Just names
kubectl get pods -o name

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

## Context and Config

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# View kubeconfig
kubectl config view
```

## Common Shortcuts

```bash
# Resource type shortcuts
po  = pods
deploy = deployments
svc = services
ns = namespaces
no = nodes
cm = configmaps
pv = persistentvolumes
pvc = persistentvolumeclaims
sa = serviceaccounts

# Example:
kubectl get po
kubectl get svc
kubectl describe deploy nginx
```

## Useful Combinations

```bash
# Get all resources
kubectl get all

# Delete all resources in namespace
kubectl delete all --all

# Get pods with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Watch deployment rollout
kubectl get pods --watch & kubectl rollout status deployment/nginx

# Force delete stuck pod
kubectl delete pod <name> --grace-period=0 --force

# Get pods sorted by restart count
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Get pod CPU/Memory
kubectl top pod <name>

# Stream logs from all pods with label
kubectl logs -f -l app=nginx --all-containers=true
```

## Tips

1. **Tab completion**: Enable kubectl autocomplete
   ```bash
   # Bash
   echo 'source <(kubectl completion bash)' >> ~/.bashrc
   
   # Zsh
   echo 'source <(kubectl completion zsh)' >> ~/.zshrc
   ```

2. **Alias for speed**:
   ```bash
   alias k=kubectl
   alias kgp='kubectl get pods'
   alias kgs='kubectl get svc'
   alias kgd='kubectl get deploy'
   ```

3. **Use `-h` for help**:
   ```bash
   kubectl -h
   kubectl get -h
   kubectl create -h
   ```

4. **Explain resources**:
   ```bash
   kubectl explain pods
   kubectl explain deployment.spec
   kubectl explain service.spec.ports
   ```

## Emergency Commands

```bash
# Cluster seems stuck
kubectl get componentstatuses

# API server not responding
kubectl cluster-info dump

# Pod stuck terminating
kubectl delete pod <name> --grace-period=0 --force

# Get all failing pods
kubectl get pods --field-selector=status.phase!=Running

# See what's consuming resources
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
```

## Pro Tips

- Use `--dry-run=client -o yaml` to generate YAML templates
- Use `kubectl diff -f file.yaml` before applying changes
- Use `kubectl wait` for scripting (wait for conditions)
- Use `kubectl proxy` for API access without auth
- Use `-o yaml > file.yaml` to backup resources

---

**Bookmark this page**. You'll reference it constantly while learning Kubernetes.
