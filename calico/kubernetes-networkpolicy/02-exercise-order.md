# Exercise: Calico NetworkPolicy Order

## Übersicht
Diese Übung demonstriert, wie die `order` Eigenschaft bei Calico NetworkPolicies die Evaluierungs-Reihenfolge bestimmt.

## Vorbereitung

### Step 1: Test-Namespace und Pods erstellen

```bash
kubectl create namespace order-test
kubectl config set-context --current --namespace=order-test
```

```bash
nano test-pods.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: order-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        role: backend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-app
  namespace: order-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-app
  template:
    metadata:
      labels:
        app: client-app
        role: frontend
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "3600"]
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: order-test
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f test-pods.yaml
```

## Step 2: Baseline Test - Verbindung funktioniert

```bash
kubectl get pods -n order-test
```

```bash
# Test: Client kann Web-App erreichen
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung erfolgreich (nginx HTML-Seite)

## Step 3: Mit GlobalNetworkPolicy starten

### Policy 1: Global Default Deny (hohe Order = später evaluiert)

```bash
nano 01-global-default-deny.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-default-deny
spec:
  order: 1000  # Hohe Order = wird zuletzt bei Global Policies evaluiert
  namespaceSelector: kubernetes.io/metadata.name == "order-test"
  selector: all()
  types:
  - Ingress
  - Egress
  ingress:
  - action: Deny
  egress:
  - action: Deny
```

```bash
kubectl apply -f 01-global-default-deny.yaml
```

### Test nach Global Default Deny

```bash
# Test: Sollte jetzt fehlschlagen
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung blockiert (Timeout)

### Policy 2: Global Allow für Frontend (mittlere Order)

```bash
nano 02-global-allow-frontend.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-allow-frontend-egress
spec:
  order: 500  # Mittlere Order = wird vor Global Default Deny evaluiert
  namespaceSelector: kubernetes.io/metadata.name == "order-test"
  selector: role == 'frontend'
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      selector: role == 'backend'
      ports: [80]
```

```bash
kubectl apply -f 02-global-allow-frontend.yaml
```

### Policy 3: Global Allow für Backend Ingress (mittlere Order)

```bash
nano 03-global-allow-backend.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-allow-backend-ingress
spec:
  order: 500  # Mittlere Order = wird vor Global Default Deny evaluiert
  namespaceSelector: kubernetes.io/metadata.name == "order-test"
  selector: role == 'backend'
  types:
  - Ingress
  ingress:
  - action: Allow
    source:
      selector: role == 'frontend'
    destination:
      ports: [80]
```

```bash
kubectl apply -f 03-global-allow-backend.yaml
```

### Test nach Global Allow-Policies

```bash
# Test: Sollte wieder funktionieren
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung erfolgreich (Global Allow-Policies mit Order 500 überschreiben Global Deny mit Order 1000)

## Step 4: Global Override mit niedriger Order demonstrieren

### Policy 4: Global Emergency Deny (niedrigste Order = zuerst evaluiert)

```bash
nano 04-global-emergency-deny.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-emergency-deny
spec:
  order: 100  # Niedrigste Order = wird zuerst bei Global Policies evaluiert
  namespaceSelector: kubernetes.io/metadata.name == "order-test"
  selector: role == 'frontend'
  types:
  - Egress
  egress:
  - action: Deny
    destination:
      ports: [80]
```

```bash
kubectl apply -f 04-global-emergency-deny.yaml
```

### Test nach Global Emergency Deny

```bash
# Test: Sollte wieder fehlschlagen
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung blockiert (Global Emergency Deny mit Order 100 hat Vorrang vor Global Allow mit Order 500)

## Step 5: Global Policy-Reihenfolge verstehen

### Aktuelle Global Policies anzeigen

```bash
# Alle Global Policies mit Order anzeigen
kubectl get globalnetworkpolicy -o custom-columns=NAME:.metadata.name,ORDER:.spec.order
```

### Evaluierungs-Reihenfolge (nur Global Policies):
1. **Order 100:** global-emergency-deny (DENY für Port 80)
2. **Order 500:** global-allow-frontend-egress (ALLOW für Backend)
3. **Order 500:** global-allow-backend-ingress (ALLOW von Frontend)
4. **Order 1000:** global-default-deny (DENY all)

**Resultat:** Global Emergency Deny (Order 100) gewinnt → Verbindung blockiert

## Step 6: Global Override mit noch niedrigerer Order

### Policy 5: Global Super Emergency Allow (niedrigste Order)

```bash
nano 05-global-super-emergency-allow.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-super-emergency-allow
spec:
  order: 50  # Niedrigste Order = allererste Evaluation bei Global Policies
  namespaceSelector: kubernetes.io/metadata.name == "order-test"
  selector: role == 'frontend'
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      selector: role == 'backend'
      ports: [80]
```

```bash
kubectl apply -f 05-global-super-emergency-allow.yaml
```

### Test nach Global Super Emergency Allow

```bash
# Test: Sollte wieder funktionieren
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung erfolgreich (Global Super Emergency Allow mit Order 50 überschreibt alle anderen Global Policies)

### Global Policy Order:
1. **Order 50:** global-super-emergency-allow (ALLOW) ← **GEWINNT**
2. **Order 100:** global-emergency-deny (DENY)
3. **Order 500:** global-allow-policies (ALLOW)
4. **Order 1000:** global-default-deny (DENY)

## Step 7: Jetzt Namespace NetworkPolicies hinzufügen

### Wichtiges Verhalten testen: Global vs Namespace Order

```bash
nano 06-namespace-super-allow.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: namespace-super-allow
  namespace: order-test
spec:
  order: 1  # Niedrigste mögliche Order bei Namespace Policies
  selector: role == 'frontend'
  types:
  - Egress
  egress:
  - action: Allow
    destination:
      selector: role == 'backend'
      ports: [80]
```

```bash
kubectl apply -f 06-namespace-super-allow.yaml
```

### Test: Global Policy überschreibt lokale Policy

```bash
# Test: Verbindung sollte funktionieren (Global Allow mit Order 50 gewinnt)
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung erfolgreich - **Global Policy mit Order 50 überschreibt Namespace Policy mit Order 1!**

### Warum? Evaluierungs-Reihenfolge:

```
GLOBAL POLICIES (werden IMMER zuerst evaluiert):
1. GlobalNetworkPolicy global-super-emergency-allow (order: 50)  ← GEWINNT
2. GlobalNetworkPolicy global-emergency-deny (order: 100)
3. GlobalNetworkPolicy global-allow-policies (order: 500)
4. GlobalNetworkPolicy global-default-deny (order: 1000)

NAMESPACE POLICIES (werden DANACH evaluiert):
5. NetworkPolicy namespace-super-allow (order: 1)               ← Wird nie erreicht!
```

### Namespace Policy mit Global Conflict

```bash
nano 07-namespace-deny.yaml
```

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: namespace-deny-all
  namespace: order-test
spec:
  order: 1  # Niedrigste Order bei Namespace Policies
  selector: role == 'frontend'
  types:
  - Egress
  egress:
  - action: Deny
    destination:
      ports: [80]
```

```bash
kubectl apply -f 07-namespace-deny.yaml
```

### Test: Global Policy gewinnt immer

```bash
# Test: Sollte immer noch funktionieren
kubectl exec -it deployment/client-app -n order-test -- wget -O- --timeout=5 http://web-service
```

**Erwartung:** Verbindung erfolgreich (Global Allow Order 50 schlägt Namespace Deny Order 1)

### Finale Evaluierungs-Reihenfolge:

```
GLOBAL POLICIES (werden immer zuerst evaluiert):
1. GlobalNetworkPolicy global-super-emergency-allow (order: 50)  ← GEWINNT
2. GlobalNetworkPolicy global-emergency-deny (order: 100)
3. GlobalNetworkPolicy global-allow-policies (order: 500)
4. GlobalNetworkPolicy global-default-deny (order: 1000)

NAMESPACE POLICIES (werden danach evaluiert, aber nie erreicht):
5. NetworkPolicy namespace-deny-all (order: 1)
6. NetworkPolicy namespace-super-allow (order: 1)
```

## Step 8: Cleanup

```bash
# Alle Global Policies löschen
kubectl delete globalnetworkpolicy global-default-deny
kubectl delete globalnetworkpolicy global-allow-frontend-egress
kubectl delete globalnetworkpolicy global-allow-backend-ingress
kubectl delete globalnetworkpolicy global-emergency-deny
kubectl delete globalnetworkpolicy global-super-emergency-allow

# Namespace löschen (löscht auch alle NetworkPolicies)
kubectl delete namespace order-test
```

## Zusammenfassung

- **Niedrige Order-Zahlen werden ZUERST evaluiert**
- **GlobalNetworkPolicy wird IMMER vor NetworkPolicy evaluiert** (unabhängig von Order!)
- **Order gilt nur innerhalb des Policy-Typs** (Global oder Namespace)
- **Erste matchende Regel gewinnt**
- **Order ermöglicht präzise Kontrolle über Policy-Hierarchien**
- **Nützlich für Override-Patterns und Emergency-Scenarios**

### Wichtige Erkenntnis:
```
GlobalNetworkPolicy (order: 9999) schlägt NetworkPolicy (order: 1)
```