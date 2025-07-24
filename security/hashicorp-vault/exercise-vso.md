# Hashicorp Vault (VSO) 

# Complete HashiCorp Vault Setup Exercise with Kubernetes, Helm, and Vault Secrets Operator

## Step 1: Create Project Structure

```bash
# Create the project directory
cd ~
mkdir -p vault-k8s-exercise
cd vault-k8s-exercise

# Create organized directory structure
mkdir -p manifests/vault
mkdir -p manifests/vault-secrets-operator
mkdir -p manifests/applications
mkdir -p scripts

# Navigate to the main directory
cd manifests/vault
```

## Step 2: Create Vault Development Configuration

```bash
nano vault-dev-values.yaml
```

**File content for `vault-dev-values.yaml`:**
```yaml
# vault-dev-values.yaml
global:
  enabled: true

server:
  # Development mode - DO NOT use in production
  dev:
    enabled: true
    devRootToken: "myroot"
  
  # Use standalone mode for development
  standalone:
    enabled: true
  
  # Disable high availability for dev
  ha:
    enabled: false
  
  # Resource limits for development
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m
  
  # LoadBalancer service configuration
  service:
    enabled: true
    type: LoadBalancer
    port: 8200
    # Optional: restrict access to internal networks
    loadBalancerSourceRanges:
      - "10.0.0.0/8"
      - "192.168.0.0/16"
      - "172.16.0.0/12"

# Enable Vault UI with LoadBalancer
ui:
  enabled: true
  serviceType: "LoadBalancer"
  
# Disable injector for this exercise
injector:
  enabled: false

# Server affinity (optional, for node selection)
serverTelemetry:
  prometheusOperator: false
```

## Step 3: Add HashiCorp Helm Repository and Install Vault

```bash
# Add HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Create vault namespace
kubectl create namespace vault

# Install Vault using our configuration
helm install vault hashicorp/vault \
  --namespace vault \
  --values vault-dev-values.yaml

# Wait for deployment to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault -n vault --timeout=300s
```

## Step 4: Verify Vault Installation

```bash
# Check vault pods
kubectl get pods -n vault

# Check vault service and get external IP
kubectl get svc -n vault

# Wait for external IP assignment
echo "Waiting for LoadBalancer IP assignment..."
kubectl get svc vault -n vault -w
```

## Step 5: Create Vault Access Script

```bash
cd ../../scripts
nano vault-access.sh
```

**File content for `vault-access.sh`:**
```bash
#!/bin/bash

# Get Vault external IP
VAULT_EXTERNAL_IP=$(kubectl get svc vault -n vault -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

if [ -z "$VAULT_EXTERNAL_IP" ]; then
    echo "LoadBalancer IP not yet assigned. Checking for hostname..."
    VAULT_EXTERNAL_IP=$(kubectl get svc vault -n vault -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
fi

if [ -z "$VAULT_EXTERNAL_IP" ]; then
    echo "External IP/Hostname not available. Using port-forward instead..."
    echo "Run: kubectl port-forward -n vault svc/vault 8200:8200"
    export VAULT_ADDR=http://localhost:8200
else
    echo "Vault External IP/Hostname: $VAULT_EXTERNAL_IP"
    export VAULT_ADDR=http://$VAULT_EXTERNAL_IP:8200
fi

export VAULT_TOKEN=myroot

echo "Vault Address: $VAULT_ADDR"
echo "Vault Token: $VAULT_TOKEN"
echo ""
echo "Testing Vault connection..."
vault status

echo ""
echo "Vault UI available at: $VAULT_ADDR"
echo "Login with token: $VAULT_TOKEN"
```

```bash
# Make script executable
chmod +x vault-access.sh

# Run the script to test Vault access
./vault-access.sh
```

## Step 6: Configure Vault Secrets

```bash
nano vault-setup.sh
```

**File content for `vault-setup.sh`:**
```bash
#!/bin/bash

# Source the access script
source ./vault-access.sh

echo "Setting up Vault secrets..."

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Create application secrets
vault kv put secret/webapp/config \
  username="admin" \
  password="supersecret123" \
  api_key="abc123xyz" \
  database_url="postgresql://user:pass@db:5432/webapp"

vault kv put secret/webapp/database \
  host="postgres.default.svc.cluster.local" \
  port="5432" \
  username="webapp_user" \
  password="db_password_123"

# Create API secrets
vault kv put secret/api/tokens \
  github_token="ghp_xxxxxxxxxxxx" \
  slack_webhook="https://hooks.slack.com/services/xxx" \
  jwt_secret="my-super-secret-jwt-key"

echo "Vault secrets created successfully!"
echo ""
echo "You can verify by running:"
echo "vault kv get secret/webapp/config"
echo "vault kv get secret/webapp/database"
echo "vault kv get secret/api/tokens"
```

```bash
# Make script executable and run it
chmod +x vault-setup.sh
./vault-setup.sh
```

## Step 7: Install Vault Secrets Operator

```bash
cd ../manifests/vault-secrets-operator
nano vso-installation.yaml
```

**File content for `vso-installation.yaml`:**
```yaml
# This file documents the VSO installation
# Run the following commands to install:

# helm install vault-secrets-operator hashicorp/vault-secrets-operator \
#   --namespace vault-secrets-operator-system \
#   --create-namespace \
#   --set "defaultVaultConnection.enabled=true" \
#   --set "defaultVaultConnection.address=http://vault.vault.svc.cluster.local:8200" \
#   --set "defaultVaultConnection.skipTLSVerify=true"
```

```bash
# Install Vault Secrets Operator
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --namespace vault-secrets-operator-system \
  --create-namespace

# Wait for VSO to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault-secrets-operator -n vault-secrets-operator-system --timeout=300s
```

## Step 8: Configure Vault Connection for VSO

```bash
nano vault-connection.yaml
```

**File content for `vault-connection.yaml`:**
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: vault-secrets-operator-system
  name: default
spec:
  # Use internal service address
  address: http://vault.vault.svc.cluster.local:8200
  
  # Skip TLS verification for dev mode
  skipTLSVerify: true
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  namespace: vault-secrets-operator-system
  name: default
spec:
  vaultConnectionRef: default
  method: token
  token:
    # Reference to secret containing the root token
    secretRef:
      name: vault-token
      key: token
---
apiVersion: v1
kind: Secret
metadata:
  namespace: vault-secrets-operator-system
  name: vault-token
type: Opaque
data:
  # Base64 encoded "myroot"
  token: bXlyb290
```

```bash
# Apply the Vault connection configuration
kubectl apply -f vault-connection.yaml

# Verify the connection
kubectl get vaultconnection -n vault-secrets-operator-system
kubectl get vaultauth -n vault-secrets-operator-system
```

## Step 9: Create VaultStaticSecret Resources

```bash
nano webapp-vault-secrets.yaml
```

**File content for `webapp-vault-secrets.yaml`:**
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: webapp-config
  namespace: default
spec:
  vaultAuthRef: vault-secrets-operator-system/default
  
  # Mount path in Vault
  mount: secret
  
  # Path to the secret
  path: webapp/config
  
  # Secret type
  type: kv-v2
  
  # Kubernetes secret to create
  destination:
    create: true
    name: webapp-secret
    type: Opaque
  
  # Refresh interval
  refreshAfter: 30s
  
  # Rollout restart for deployments using this secret
  rolloutRestartTargets:
    - kind: Deployment
      name: webapp
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: webapp-database
  namespace: default
spec:
  vaultAuthRef: vault-secrets-operator-system/default
  
  mount: secret
  path: webapp/database
  type: kv-v2
  
  destination:
    create: true
    name: webapp-db-secret
    type: Opaque
  
  refreshAfter: 30s
  
  rolloutRestartTargets:
    - kind: Deployment
      name: webapp
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: api-tokens
  namespace: default
spec:
  vaultAuthRef: vault-secrets-operator-system/default
  
  mount: secret
  path: api/tokens
  type: kv-v2
  
  destination:
    create: true
    name: api-tokens-secret
    type: Opaque
  
  refreshAfter: 30s
```

```bash
# Apply the VaultStaticSecret resources
kubectl apply -f webapp-vault-secrets.yaml

# Check the status of VaultStaticSecrets
kubectl get vaultstaticsecret
kubectl describe vaultstaticsecret webapp-config
```

## Step 10: Create Sample Application

```bash
cd ../applications
nano webapp-deployment.yaml
```

**File content for `webapp-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        
        # Environment variables from Vault secrets
        env:
        - name: APP_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: username
        - name: APP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: api_key
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: database_url
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: host
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-db-secret
              key: password
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: api-tokens-secret
              key: github_token
        
        # Mount secrets as files
        volumeMounts:
        - name: app-secrets
          mountPath: /etc/secrets/app
          readOnly: true
        - name: db-secrets
          mountPath: /etc/secrets/db
          readOnly: true
        - name: api-secrets
          mountPath: /etc/secrets/api
          readOnly: true
        
        # Custom nginx config to display environment variables
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          
      volumes:
      - name: app-secrets
        secret:
          secretName: webapp-secret
      - name: db-secrets
        secret:
          secretName: webapp-db-secret
      - name: api-secrets
        secret:
          secretName: api-tokens-secret
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        location /env {
            add_header Content-Type text/plain;
            return 200 "Environment Variables loaded from Vault secrets\n";
        }
    }
```

```bash
# Apply the application
kubectl apply -f webapp-deployment.yaml

# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/webapp --timeout=300s
```

## Step 11: Create Verification Script

```bash
cd ../../scripts
nano verify-setup.sh
```

**File content for `verify-setup.sh`:**
```bash
#!/bin/bash

echo "=== Vault Kubernetes Exercise Verification ==="
echo ""

echo "1. Checking Vault pods..."
kubectl get pods -n vault
echo ""

echo "2. Checking Vault service..."
kubectl get svc -n vault
echo ""

echo "3. Checking Vault Secrets Operator..."
kubectl get pods -n vault-secrets-operator-system
echo ""

echo "4. Checking VaultConnection and VaultAuth..."
kubectl get vaultconnection,vaultauth -n vault-secrets-operator-system
echo ""

echo "5. Checking VaultStaticSecrets..."
kubectl get vaultstaticsecret
echo ""

echo "6. Checking generated Kubernetes secrets..."
kubectl get secrets | grep -E "(webapp-secret|webapp-db-secret|api-tokens-secret)"
echo ""

echo "7. Checking webapp deployment..."
kubectl get deployment webapp
kubectl get pods -l app=webapp
echo ""

echo "8. Checking webapp service..."
kubectl get svc webapp-service
echo ""

echo "9. Getting secret contents (base64 decoded)..."
echo "webapp-secret:"
kubectl get secret webapp-secret -o jsonpath='{.data.username}' | base64 -d && echo ""
kubectl get secret webapp-secret -o jsonpath='{.data.api_key}' | base64 -d && echo ""
echo ""

echo "webapp-db-secret:"
kubectl get secret webapp-db-secret -o jsonpath='{.data.host}' | base64 -d && echo ""
kubectl get secret webapp-db-secret -o jsonpath='{.data.username}' | base64 -d && echo ""
echo ""

echo "10. Testing Vault access..."
source ./vault-access.sh > /dev/null 2>&1
echo "Vault status:"
vault status
echo ""

echo "11. Listing Vault secrets..."
vault kv list secret/
echo ""

echo "=== Verification Complete ==="
echo ""
echo "Next steps:"
echo "- Access Vault UI at: $VAULT_ADDR (token: myroot)"
echo "- Check webapp service external IP: kubectl get svc webapp-service"
echo "- View pod logs: kubectl logs -l app=webapp"
echo "- Test secret rotation: Update secrets in Vault and watch pods restart"
```

```bash
# Make script executable and run verification
chmod +x verify-setup.sh
./verify-setup.sh
```

## Step 12: Create Cleanup Script

```bash
nano cleanup.sh
```

**File content for `cleanup.sh`:**
```bash
#!/bin/bash

echo "Cleaning up Vault Kubernetes Exercise..."

# Delete application
kubectl delete -f ../manifests/applications/webapp-deployment.yaml --ignore-not-found

# Delete VaultStaticSecrets
kubectl delete -f ../manifests/vault-secrets-operator/webapp-vault-secrets.yaml --ignore-not-found

# Delete Vault connection
kubectl delete -f ../manifests/vault-secrets-operator/vault-connection.yaml --ignore-not-found

# Uninstall Vault Secrets Operator
helm uninstall vault-secrets-operator -n vault-secrets-operator-system --ignore-not-found

# Delete VSO namespace
kubectl delete namespace vault-secrets-operator-system --ignore-not-found

# Uninstall Vault
helm uninstall vault -n vault --ignore-not-found

# Delete Vault namespace
kubectl delete namespace vault --ignore-not-found

echo "Cleanup complete!"
```

```bash
chmod +x cleanup.sh
```

## Step 13: Create README

```bash
cd ..
nano README.md
```

**File content for `README.md`:**
```markdown
# HashiCorp Vault with Kubernetes Exercise

This exercise demonstrates setting up HashiCorp Vault in development mode on Kubernetes using Helm and the Vault Secrets Operator (VSO).

## Project Structure

```
vault-k8s-exercise/
├── manifests/
│   ├── vault/
│   │   └── vault-dev-values.yaml
│   ├── vault-secrets-operator/
│   │   ├── vso-installation.yaml
│   │   ├── vault-connection.yaml
│   │   └── webapp-vault-secrets.yaml
│   └── applications/
│       └── webapp-deployment.yaml
├── scripts/
│   ├── vault-access.sh
│   ├── vault-setup.sh
│   ├── verify-setup.sh
│   └── cleanup.sh
└── README.md
```

## Prerequisites

- Kubernetes cluster with LoadBalancer support
- Helm 3.x installed
- kubectl configured
- vault CLI (optional, for testing)

## Quick Start

1. Run the verification script:
   ```bash
   cd scripts
   ./verify-setup.sh
   ```

2. Access Vault UI using the external LoadBalancer IP
3. Login with token: `myroot`

## Key Components

- **Vault**: Running in development mode with LoadBalancer service
- **Vault Secrets Operator**: Manages secrets synchronization
- **VaultStaticSecret**: CRDs that sync Vault secrets to Kubernetes secrets
- **Sample Application**: Nginx deployment using secrets from Vault

## Important Notes

- This is a **DEVELOPMENT** setup only
- Never use development mode in production
- Secrets are not persisted (stored in memory)
- TLS is disabled for simplicity

## Cleanup

To remove all resources:
```bash
cd scripts
./cleanup.sh
```
```

## Final Verification

```bash
# Navigate back to project root
cd ~/vault-k8s-exercise

# Run final verification
cd scripts
./verify-setup.sh

# Check the complete project structure
cd ..
find . -type f -name "*.yaml" -o -name "*.sh" -o -name "*.md" | sort
```

## Summary

You now have a complete HashiCorp Vault setup with:

1. ✅ **Vault** running in development mode with LoadBalancer access
2. ✅ **Vault Secrets Operator** managing secret synchronization
3. ✅ **VaultStaticSecret** resources creating Kubernetes secrets from Vault
4. ✅ **Sample application** using secrets from Vault
5. ✅ **Scripts** for setup, verification, and cleanup
6. ✅ **Organized project structure** with proper documentation

**Access Points:**
- **Vault UI**: `http://<LoadBalancer-IP>:8200` (token: `myroot`)
- **Application**: `http://<webapp-service-LoadBalancer-IP>`

**Project Location:** `~/vault-k8s-exercise/`

This exercise demonstrates a complete workflow from Vault installation to secret consumption in applications using modern Kubernetes-native approaches!
