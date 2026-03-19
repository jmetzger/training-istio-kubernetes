# Istio Canary Upgrade mit Helm und Revision Tags

## Überblick

Revision Tags sind stabile Aliase für Istio-Revisionen. Namespaces referenzieren nur den Tag-Namen – beim Upgrade wird der Tag auf die neue Revision umgeschwenkt, ohne Namespace-Labels ändern zu müssen.

```
Namespace (istio.io/rev: prod) ──► Tag "prod" ──► istiod Revision 1-22-0
                                                    ↓  (nach Upgrade)
                                                   istiod Revision 1-23-0
```

---

## Voraussetzungen

```bash
# Helm Repo hinzufügen
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# istioctl installieren (wird für Tag-Management benötigt)
curl -sL https://istio.io/downloadIstio | ISTIO_VERSION=1.22.0 sh -
export PATH=$PWD/istio-1.22.0/bin:$PATH
```

---

## Schritt 1: Initiale Installation (Revision 1-22-0)

### 1.1 Istio Base (CRDs)

```bash
helm install istio-base istio/base \
  -n istio-system \
  --create-namespace \
  --version 1.22.0
```

### 1.2 Istiod mit Revision

> **Wichtig:** Bei Revision-basierter Installation wird `--set revision=` gesetzt.
> Der Revisionsname darf nur alphanumerische Zeichen und `-` enthalten (keine Punkte).

```bash
helm install istiod-1-22-0 istio/istiod \
  -n istio-system \
  --version 1.22.0 \
  --set revision=1-22-0 \
  --wait
```

Prüfen:

```bash
kubectl get pods -n istio-system -l app=istiod
# NAME                            READY   STATUS
# istiod-1-22-0-xxxxxxxxx-xxxxx   1/1     Running
```

### 1.3 Gateway (optional)

```bash
helm install istio-ingressgateway istio/gateway \
  -n istio-ingress \
  --create-namespace \
  --version 1.22.0 \
  --set revision=1-22-0
```

---

## Schritt 2: Revision Tag erstellen

```bash
# Tag "prod" zeigt auf Revision 1-22-0
istioctl tag set prod --revision 1-22-0

# Prüfen
istioctl tag list
# TAG    REVISION
# prod   1-22-0
```

Technisch erzeugt das eine `MutatingWebhookConfiguration` namens `istio-revision-tag-prod`.

---

## Schritt 3: Namespaces labeln

```bash
# Namespace mit dem Tag (nicht der Revision!) labeln
kubectl label namespace default istio.io/rev=prod

# Falls noch das alte (nicht-revisionierte) Label existiert, entfernen!
# "istio-injection=enabled" hat Vorrang vor "istio.io/rev" und würde
# den Revision-Tag ignorieren. Nur nötig wenn vorher ohne Revisionen installiert.
kubectl label namespace default istio-injection-
```

Demo-App deployen:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl rollout status deployment -n default
```

Sidecar-Injection prüfen:

```bash
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
# Jeder Pod sollte einen "istio-proxy" Container haben
```

---

## Schritt 4: Canary Upgrade durchführen

### 4.1 Neue Istiod-Revision installieren

```bash
helm install istiod-1-23-0 istio/istiod \
  -n istio-system \
  --version 1.23.0 \
  --set revision=1-23-0 \
  --wait
```

Jetzt laufen **zwei** Istiod-Instanzen parallel:

```bash
kubectl get pods -n istio-system -l app=istiod
# NAME                             READY   STATUS
# istiod-1-22-0-xxxxxxxxx-xxxxx    1/1     Running
# istiod-1-23-0-yyyyyyyyy-yyyyy    1/1     Running
```

### 4.2 (Optional) Canary-Tag für Testphase

```bash
# Canary-Tag für einzelne Test-Namespaces
istioctl tag set canary --revision 1-23-0

# Test-Namespace umschalten
kubectl label namespace test-ns istio.io/rev=canary --overwrite
kubectl rollout restart deployment -n test-ns
```

Hier testen, ob alles mit der neuen Version funktioniert.

### 4.3 Prod-Tag auf neue Revision schwenken

```bash
# Tag "prod" zeigt jetzt auf 1-23-0
istioctl tag set prod --revision 1-23-0 --overwrite

# Prüfen
istioctl tag list
# TAG      REVISION
# prod     1-23-0
# canary   1-23-0
```

### 4.4 Workloads neu starten

Die Namespace-Labels bleiben unverändert (`istio.io/rev=prod`). Neue Pods bekommen automatisch den Sidecar der neuen Revision. Bestehende Pods müssen neu gestartet werden:

```bash
# Alle Deployments im Namespace neu starten
kubectl rollout restart deployment -n default

# Warten bis alle Pods ready sind
kubectl rollout status deployment -n default
```

### 4.5 Sidecar-Version prüfen

```bash
# Proxy-Version aller Pods anzeigen
istioctl proxy-status
# NAME                    CLUSTER   CDS   LDS   EDS   RDS   ECDS   ISTIOD                          VERSION
# productpage-v1-...      ...       SYNCED ...                      istiod-1-23-0-yyyyy.istio-system 1.23.0
```

---

## Schritt 5: Alte Revision entfernen

Erst wenn **alle** Workloads auf der neuen Revision laufen:

```bash
# Prüfen ob noch Proxies an alter Revision hängen
istioctl proxy-status | grep 1-22-0
# Sollte leer sein!

# Canary-Tag entfernen (falls erstellt)
istioctl tag remove canary

# Alte Istiod-Revision deinstallieren
helm uninstall istiod-1-22-0 -n istio-system

# Altes Gateway upgraden (falls installiert)
helm upgrade istio-ingressgateway istio/gateway \
  -n istio-ingress \
  --version 1.23.0 \
  --set revision=1-23-0

# Istio Base CRDs updaten
helm upgrade istio-base istio/base \
  -n istio-system \
  --version 1.23.0
```

---

## Rollback

Falls Probleme auftreten, Tag einfach zurückschwenken:

```bash
# Tag zurück auf alte Revision
istioctl tag set prod --revision 1-22-0 --overwrite

# Pods neu starten
kubectl rollout restart deployment -n default

# Neue Revision wieder entfernen
helm uninstall istiod-1-23-0 -n istio-system
```

---

## Zusammenfassung der Kommandos

| Schritt | Kommando |
|---------|----------|
| Neue Revision installieren | `helm install istiod-1-23-0 istio/istiod --set revision=1-23-0` |
| Tag umschalten | `istioctl tag set prod --revision 1-23-0 --overwrite` |
| Pods neu starten | `kubectl rollout restart deployment -n <namespace>` |
| Status prüfen | `istioctl proxy-status` |
| Alte Revision entfernen | `helm uninstall istiod-1-22-0` |
| Rollback | `istioctl tag set prod --revision 1-22-0 --overwrite` |
