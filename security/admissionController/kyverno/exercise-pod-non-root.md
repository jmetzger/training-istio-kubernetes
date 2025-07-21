# Übung: Pod Non-Root User Enforcement with Kyverno

## Prerequisites
- Kubernetes cluster with admission controller support
- kubectl configured with cluster admin privileges
- Helm 3 installed

## Exercise Steps

### Step 1: Create Project Directory Structure

```bash
cd
mkdir -p manifests/kyverno-non-root-pod
cd manifests/kyverno-non-root-pod
```

### Step 2: Create Values File and Install Kyverno using Helm

Create a helm directory and values file for high availability configuration:

```bash
mkdir helm
nano helm/values.yaml
```

```
admissionController:
  replicas: 3
backgroundController:
  replicas: 3
cleanupController:
  replicas: 3
reportsController:
  replicas: 3
```

```bash
# Add Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Install Kyverno in kyverno namespace with custom values
helm upgrade --install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --version 3.4.4 \
  --values helm/values.yaml \
  --wait

# Verify installation (should show 3 replicas for each controller)
# Wichtig: Das dauert einen Moment 
kubectl get pods -n kyverno
kubectl get deployment -n kyverno
```

### Step 3: Create Kyverno Policy for Non-Root Enforcement

Create the policy file:

```bash
nano non-root-policy.yaml
```

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root-user
  annotations:
    policies.kyverno.io/title: Require Non-Root User
    policies.kyverno.io/category: Pod Security Standards (Restricted)
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      Containers must run as non-root users. This policy ensures that the securityContext.runAsUser 
      field is set to a value greater than 0 and securityContext.runAsNonRoot is set to true.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: check-non-root-user
    match:
      any:
      - resources:
          kinds:
          - Pod
    exclude:
        resources:
          namespaces:
            - kube-system
            - monitoring

    validate:
      message: "Pod must run as non-root user (runAsUser > 0 and runAsNonRoot: true)"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
            runAsUser: ">0"
          containers:
          - securityContext:
              runAsNonRoot: true
              runAsUser: ">0"
```

### Step 4: Apply the Policy

```bash
kubectl apply -f non-root-policy.yaml

# Verify policy is created
kubectl get clusterpolicy
kubectl describe clusterpolicy require-non-root-user
```

### Step 5: Test with Root User Pod (Should Fail)

Create a test pod that should be rejected:

```bash
nano test-root-pod.yaml
```

```
# Variante: nichts ist gesetzt 
apiVersion: v1
kind: Pod
metadata:
  name: test-root-pod
  namespace: default
spec:
  securityContext:
    runAsUser: 0
    runAsNonRoot: false
  containers:
  - name: nginx
    image: nginx:1.27.0
    securityContext:
      runAsUser: 0
      runAsNonRoot: false
```

Try to apply it (should fail):

```bash
kubectl apply -f .
```

```bash
nano test-root-pod-v2.yaml
```

```
# Variante: nichts ist gesetzt 
apiVersion: v1
kind: Pod
metadata:
  name: test-root-pod-v2
  namespace: default
spec:
  securityContext:
    runAsUser: 0
    runAsNonRoot: false
  containers:
  - name: nginx
    image: nginx:1.27.0
    securityContext:
      runAsUser: 0
      runAsNonRoot: false
```

```bash 
kubectl apply -f .
```

### Step 6: Test with Non-Root Pod (Should Succeed)

Create a pod that complies with the policy:

```bash
nano test-non-root-pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: test-non-root-pod
  namespace: default
spec:
  securityContext:
    runAsUser: 1001
    runAsNonRoot: true
    runAsGroup: 1001
    fsGroup: 1001
  containers:
  - name: nginx
    image: bitnami/nginx:1.27.0
    securityContext:
      runAsUser: 1001
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false
```

```bash
kubectl apply -f test-non-root-pod.yaml
```

```
# Verify the pod is running
kubectl get pod test-non-root-pod
kubectl exec test-non-root-pod -- id
```

## Verification and Cleanup

```bash
# Check policy violations
kubectl get events --field-selector reason=PolicyViolation

# View Kyverno admission controller logs if needed
kubectl logs -n kyverno -l app.kubernetes.io/name=kyverno

# Cleanup test pods
kubectl delete -f . 
```

## Expected Results

1. **Root Pod Test**: Should fail with policy violation message about requiring non-root user
2. **Non-Root Pod Test**: Should succeed and run with UID 1001
3. **Policy Verification**: Kyverno should log admission decisions

## Notes

- Kyverno provides more flexible policy management than Pod Security Standards
- The `bitnami/nginx` image is designed to run as non-root user by default
- Regular `nginx` image runs as root by default, hence the failure
- Policies can be tested in `Audit` mode before enforcing them
