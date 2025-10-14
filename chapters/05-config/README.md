# Chapter 5: ConfigMaps & Secrets - Configuration Done Right

You've deployed apps, scaled them, exposed them. But all configuration is hardcoded in YAML. Time to separate config from code like a professional.

## The Restaurant Menu Analogy

Imagine you run a restaurant chain with locations in different cities:

- **Jakarta branch**: Prices in Rupiah, spicy food by default, Halal menu
- **New York branch**: Prices in USD, mild by default, diverse menu
- **Tokyo branch**: Prices in Yen, umami flavors, Japanese-style service

Same restaurant concept, **different configurations** per location.

You don't print the menu into the kitchen equipment. You have:
- **Menu boards** (ConfigMaps) - Public info like prices, items, hours
- **Safe combination** (Secrets) - Private info like credit card terminals, alarm codes

**Kubernetes ConfigMaps and Secrets** work the same way:
- **ConfigMaps** = Non-sensitive configuration data
- **Secrets** = Sensitive data like passwords, API keys, tokens

## The Reality: Why Separate Config?

**Bad approach (hardcoded):**
```yaml
env:
- name: DATABASE_HOST
  value: "prod-db.example.com"  # Hardcoded!
- name: API_KEY
  value: "sk_live_abc123xyz"    # Exposed in Git!
```

**Problems:**
- Can't reuse same deployment for dev/staging/prod
- Secrets exposed in version control
- Need to rebuild container for config changes
- No separation of concerns (dev vs ops)

**Good approach (externalized):**
```yaml
env:
- name: DATABASE_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: db_host
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: api_key
```

**Benefits:**
- Same container, different configs
- Secrets encrypted at rest
- Change config without rebuilding
- Developers don't see production secrets

## ConfigMaps: Non-Sensitive Configuration

### Experiment 1: Create ConfigMap from Literals

**Create a ConfigMap:**
```bash
kubectl create configmap app-config \
  --from-literal=app_name="My Awesome App" \
  --from-literal=log_level="debug" \
  --from-literal=max_connections="100"

# Check it
kubectl get configmap app-config

# See the data
kubectl describe configmap app-config
```

**Output shows:**
```
Data
====
app_name:
----
My Awesome App
log_level:
----
debug
max_connections:
----
100
```

### Experiment 2: Use ConfigMap in Pod (Environment Variables)

**Create pod using ConfigMap:**

```yaml
# File: pod-with-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "App: $APP_NAME, LogLevel: $LOG_LEVEL"; sleep 3600']
    env:
    - name: APP_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app_name
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
```

```bash
kubectl apply -f pod-with-configmap.yaml

# Check logs
kubectl logs app-pod
# Output: App: My Awesome App, LogLevel: debug
```

**What happened?**
- ConfigMap data injected as environment variables
- Pod reads config without hardcoding
- Change ConfigMap, restart pod, new config applied

### Experiment 3: Use ConfigMap as Volume (File Mount)

Sometimes you need config as files (like nginx.conf, application.properties).

**Create ConfigMap from file:**

```bash
# Create a config file
cat > app.properties << EOF
database.host=localhost
database.port=5432
cache.enabled=true
cache.ttl=300
EOF

# Create ConfigMap from file
kubectl create configmap app-properties --from-file=app.properties

# Verify
kubectl describe configmap app-properties
```

**Mount as volume in pod:**

```yaml
# File: pod-with-config-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config-file
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'cat /etc/config/app.properties && sleep 3600']
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-properties
```

```bash
kubectl apply -f pod-with-config-volume.yaml

# Check logs
kubectl logs app-with-config-file
# Shows content of app.properties
```

**Use case:** Perfect for config files that apps read on startup.

### Experiment 4: ConfigMap from YAML

For better version control, define ConfigMaps in YAML:

```yaml
# File: app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-yaml
data:
  app_name: "Production App"
  log_level: "info"
  feature_flags: |
    {
      "new_ui": true,
      "beta_features": false,
      "analytics": true
    }
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

```bash
kubectl apply -f app-configmap.yaml

kubectl get configmap app-config-yaml -o yaml
```

**Notice:**
- Simple key-value pairs
- Multi-line values using `|`
- Can store JSON, YAML, config files, etc.

## Secrets: Sensitive Data

Secrets are like ConfigMaps but for sensitive data. Kubernetes stores them base64-encoded (not encrypted by default, but better than plaintext).

### Experiment 5: Create Secret from Literals

```bash
kubectl create secret generic app-secrets \
  --from-literal=db_password="super_secret_pwd" \
  --from-literal=api_key="sk_live_abc123xyz456"

# Check it
kubectl get secrets

# Describe (data is hidden)
kubectl describe secret app-secrets
```

**Output:**
```
Data
====
api_key:      23 bytes
db_password:  16 bytes
```

Notice: Values not shown in describe (security).

**View actual values:**
```bash
kubectl get secret app-secrets -o yaml
```

You'll see base64-encoded values:
```yaml
data:
  api_key: c2tfbGl2ZV9hYmMxMjN4eXo0NTY=
  db_password: c3VwZXJfc2VjcmV0X3B3ZA==
```

**Decode:**
```bash
echo "c3VwZXJfc2VjcmV0X3B3ZA==" | base64 --decode
# Output: super_secret_pwd
```

**Important:** Base64 is encoding, NOT encryption. Anyone with cluster access can decode.

### Experiment 6: Use Secrets in Pod

```yaml
# File: pod-with-secrets.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "DB Password: $DB_PASSWORD"; sleep 3600']
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db_password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api_key
```

```bash
kubectl apply -f pod-with-secrets.yaml

kubectl logs app-with-secrets
# Output: DB Password: super_secret_pwd
```

### Experiment 7: Mount Secrets as Files

Useful for certificate files, SSH keys, etc.

```bash
# Create secret from file
echo "my-super-secret-token" > token.txt
kubectl create secret generic token-secret --from-file=token.txt

# Use in pod
cat > pod-with-secret-file.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret-file
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'cat /etc/secrets/token.txt && sleep 3600']
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: token-secret
EOF

kubectl apply -f pod-with-secret-file.yaml
kubectl logs app-with-secret-file
```

## ConfigMap vs Secret: When to Use What?

| Use ConfigMap | Use Secret |
|---------------|------------|
| Database host/port | Database password |
| API endpoints | API keys/tokens |
| Feature flags | TLS certificates |
| Log levels | SSH keys |
| App settings | OAuth credentials |
| Public configuration | Private keys |

**Rule of thumb:** If you'd put it in `.env` file and add to `.gitignore`, use Secret.

## Real-World Example: Multi-Environment Deployment

**Scenario:** Same app, different configs for dev/staging/prod.

**ConfigMaps:**

```yaml
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  environment: "development"
  api_url: "http://dev-api.internal"
  debug: "true"
---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  environment: "production"
  api_url: "https://api.example.com"
  debug: "false"
```

**Deployment (same for both namespaces):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v1.0
        env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: api_url
```

**Deploy:**
```bash
# Dev
kubectl apply -f dev-config.yaml -n dev
kubectl apply -f deployment.yaml -n dev

# Prod
kubectl apply -f prod-config.yaml -n prod
kubectl apply -f deployment.yaml -n prod
```

Same deployment YAML, different behavior based on ConfigMap.

## Updating ConfigMaps and Secrets

**Important:** Changing ConfigMap/Secret doesn't auto-restart pods.

**Option 1: Manual restart**
```bash
# Update ConfigMap
kubectl create configmap app-config --from-literal=log_level="info" --dry-run=client -o yaml | kubectl apply -f -

# Restart deployment to pick up changes
kubectl rollout restart deployment/app
```

**Option 2: Use volume mounts**
- Mounted ConfigMaps update automatically (may take 60s)
- App needs to watch file for changes and reload

**Option 3: Immutable ConfigMaps (Recommended)**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2  # New version
immutable: true
data:
  log_level: "info"
```

- Create new ConfigMap with version suffix
- Update deployment to use new ConfigMap
- Rolling update applies new config
- Old ConfigMap stays until all pods updated

## Common Mistakes

### Mistake 1: Putting Secrets in Git

**Wrong:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
stringData:
  password: "actual_password_here"  # In Git! Bad!
```

**Better:**
- Use `kubectl create secret` command
- Use secret management tools (Sealed Secrets, External Secrets Operator)
- Store encrypted in Git with tools like SOPS

### Mistake 2: Forgetting to Restart Pods

```bash
# You update ConfigMap
kubectl edit configmap app-config

# But pods still use old config!
# Need to restart:
kubectl rollout restart deployment/app
```

### Mistake 3: Wrong Key Reference

```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: loglevel  # Typo! Should be "log_level"
```

**Result:** Pod fails to start with error "key not found".

## The Challenge

Create a web application with proper config separation:

**Requirements:**
1. **Deployment** running nginx
2. **ConfigMap** with custom nginx.conf that:
   - Serves on port 8080
   - Has custom server name
   - Proxies requests to a backend
3. **Secret** with basic auth credentials
4. Mount ConfigMap as file at `/etc/nginx/nginx.conf`
5. Use Secret for authentication

**Hints:**
```yaml
# Basic auth nginx config
location / {
  auth_basic "Restricted";
  auth_basic_user_file /etc/nginx/.htpasswd;
}
```

**Try it yourself.**

<details>
<summary>Solution (click to expand)</summary>

**configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 8080;
        server_name myapp.local;
        
        location / {
          auth_basic "Restricted Area";
          auth_basic_user_file /etc/nginx/.htpasswd;
          root /usr/share/nginx/html;
        }
      }
    }
```

**secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-auth
type: Opaque
stringData:
  .htpasswd: |
    admin:$apr1$abc123$xyz456789
```

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: auth
          mountPath: /etc/nginx/.htpasswd
          subPath: .htpasswd
      volumes:
      - name: config
        configMap:
          name: nginx-config
      - name: auth
        secret:
          secretName: nginx-auth
```

**Deploy:**
```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml

# Test
kubectl port-forward deployment/nginx-app 8080:8080
curl -u admin:password http://localhost:8080
```

</details>

## Cleanup

```bash
kubectl delete configmap app-config app-properties app-config-yaml
kubectl delete secret app-secrets token-secret
kubectl delete pod app-pod app-with-config-file app-with-secrets app-with-secret-file
```

## Key Takeaways

1. **Separate config from code** - Use ConfigMaps and Secrets, not hardcoded values
2. **ConfigMaps for non-sensitive** - Database hosts, API endpoints, feature flags
3. **Secrets for sensitive** - Passwords, API keys, certificates
4. **Base64 is not encryption** - Secrets are encoded, not secure without additional measures
5. **Restart pods for updates** - ConfigMap/Secret changes don't auto-propagate to env vars
6. **Never commit secrets to Git** - Use secret management tools or manual creation

## What You Can Do Now

- Externalize application configuration
- Use environment variables from ConfigMaps
- Mount config files into containers
- Store sensitive data in Secrets
- Manage multi-environment deployments
- Update config without rebuilding containers

## What's Next

You can configure apps properly. But what if pods restart and lose data? Time to learn persistent storage.

**Next**: Chapter 6: Persistent Storage (Coming Soon)

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Key command**: `kubectl create configmap` and `kubectl create secret` - Your config management tools
