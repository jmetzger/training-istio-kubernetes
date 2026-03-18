# Übung: Säule 1 – mTLS prüfen und aktivieren/deaktivieren

## Ziel

mTLS-Status im Mesh inspizieren, PeerAuthentication-Policies anwenden und die Auswirkungen auf den Traffic anhand der Bookinfo-Anwendung verifizieren.

---

## Voraussetzungen

- Kubernetes-Cluster mit Istio (Sidecar- oder Ambient-Mode)
- `istioctl` installiert
- Namespace `bookinfo` mit Istio-Injection:

```bash
kubectl create ns bookinfo
kubectl label ns bookinfo istio-injection=enabled
```

---

## Schritt 1: Bookinfo deployen

   * Bereit deployed

**Architektur-Überblick:**

```
productpage → details
            → reviews-v1 (keine Sterne)
            → reviews-v2 → ratings (schwarze Sterne)
            → reviews-v3 → ratings (rote Sterne)
```

### Funktionstest

```bash
kubectl -n bookinfo exec deploy/productpage-v1 -- \
  curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

Erwartete Ausgabe: `<title>Simple Bookstore App</title>`

---

## Schritt 2: Aktuellen mTLS-Status prüfen

### 2a) Mit istioctl

```bash
# mTLS-Status der productpage prüfen
istioctl x describe pod \
  $(kubectl -n bookinfo get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  -n bookinfo

# mTLS-Status von ratings prüfen
istioctl x describe pod \
  $(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
  -n bookinfo
```

**Frage:** Welchen mTLS-Mode zeigt Istio standardmäßig an?

### 2b) Bestehende PeerAuthentication-Policies anzeigen

```bash
# Mesh-weit (istio-system)
kubectl get peerauthentication -A

# Im Namespace
kubectl get peerauthentication -n bookinfo
```

> **Hinweis:** PeerAuthentication ist eine **Namespaced Resource** (`NAMESPACED=true`).
> Die Geltung hängt davon ab, *wo* sie erstellt wird:
>
> - Im **App-Namespace** (z.B. `bookinfo`) → gilt nur für diesen Namespace
> - In **`istio-system`** → wirkt als Mesh-weite Policy (quasi cluster-weit)
>
> Es gibt keine echte Cluster-Resource – die „globale" Wirkung entsteht nur durch die Konvention, dass Istio eine PeerAuthentication in `istio-system` als Mesh-Default behandelt.
>
> ```bash
> kubectl api-resources | grep peerauthentication
> # peerauthentications   pa   security.istio.io/v1   true   PeerAuthentication
> ```
>
> **Wichtig:** Wenn weder im App-Namespace noch in `istio-system` eine PeerAuthentication gesetzt ist, zeigt `kubectl get peerauthentication -A` ein leeres Ergebnis (`No resources found`). In diesem Fall greift automatisch **PERMISSIVE** – dieser Default ist im Istio-Quellcode hardcoded und nicht über die meshConfig konfigurierbar. Prüfen lässt sich der effektive Mode mit:
>
> ```bash
> istioctl x describe pod <pod-name> -n bookinfo
> # Ausgabe enthält: "mTLS mode: PERMISSIVE"
> ```

### 2c) Verifizierung über Request-Header

```bash
# sleep als Debug-Client deployen
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml
kubectl -n bookinfo wait --for=condition=ready pod -l app=sleep --timeout=60s

# Header bei ratings prüfen – XFCC-Header zeigt mTLS-Identität
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -v http://ratings:9080/ratings/0 2>&1 | grep -i x-forwarded-client-cert
```

**Frage:** Welche `X-Forwarded-Client-Cert`-Header sehen Sie? Was bedeutet das?

---

## Schritt 3: mTLS auf STRICT setzen

### 3a) Namespace-weite STRICT Policy

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

### 3b) Verifizieren – Kommunikation im Mesh funktioniert weiterhin

```bash
# Falls nicht vorher schon erstellt 
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml

# productpage kann weiterhin details und reviews erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage:9080/productpage

# sleep kann ratings erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

### 3c) Zugriff ohne Sidecar testen

Starten Sie einen Pod **ohne** Sidecar und versuchen Sie, die Bookinfo-Services zu erreichen:

```bash
# Pod ohne Sidecar in einem Namespace ohne Injection
kubectl create ns no-mesh
kubectl -n no-mesh apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml
kubectl -n no-mesh wait --for=condition=ready pod --all --timeout=60s

# Zugriff auf productpage versuchen
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" productpage.bookinfo:9080/productpage

# Zugriff auf ratings versuchen
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" ratings.bookinfo:9080/ratings/0
```

**Erwartetes Ergebnis:** Die Verbindungen schlagen fehl, da der Client kein mTLS-Zertifikat präsentiert. curl gibt den HTTP-Statuscode 000 aus (keine HTTP-Antwort) und bricht mit Exit Code 56 (CURLE_RECV_ERROR) ab – Envoy resettet die Verbindung bereits auf TLS-Ebene, bevor es zur HTTP-Kommunikation kommt.

---

```
kubectl delete ns no-mesh
```

## Schritt 4: mTLS auf PERMISSIVE setzen

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

### Verifizieren

```bash
# Aus dem Mesh – funktioniert (mit mTLS)
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" productpage:9080/productpage

# Von außerhalb des Mesh – funktioniert jetzt auch (ohne mTLS)
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" productpage.bookinfo:9080/productpage
```

**Frage:** Warum ist PERMISSIVE der Istio-Default und wann ist STRICT besser?

---

##

## Schritt 5: Workload-spezifisches mTLS

mTLS lässt sich gezielt für einzelne Services konfigurieren. Hier setzen wir nur den ratings-Service auf STRICT, während der Rest PERMISSIVE bleibt:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: ratings-strict
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  mtls:
    mode: STRICT
EOF
```

### Verifizieren

```bash
# ratings ist jetzt STRICT – von außerhalb nicht erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" ratings.bookinfo:9080/ratings/0

# productpage ist weiterhin PERMISSIVE – von außerhalb erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" productpage.bookinfo:9080/productpage

# reviews (im Mesh) kann ratings weiterhin erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}" ratings:9080/ratings/0
```

**Frage:** In welcher Reihenfolge wertet Istio PeerAuthentication-Policies aus?

```bash
# Aufräumen
kubectl -n bookinfo delete peerauthentication ratings-strict
```

---

## Schritt 6: Port-Level mTLS (Bonus)

mTLS lässt sich auch pro Port konfigurieren:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: ratings-per-port
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  mtls:
    mode: STRICT
  portLevelMtls:
    9080:
      mode: PERMISSIVE
EOF
```

**Frage:** Für welches Szenario ist Port-Level-Konfiguration sinnvoll?

```bash
kubectl -n bookinfo delete peerauthentication ratings-per-port
```

---

## Schritt 7: mTLS mit DestinationRule steuern

Die **Client-Seite** wird über DestinationRules konfiguriert:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ratings-mtls
  namespace: bookinfo
spec:
  host: ratings.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

**Hinweis:** Seit Istio 1.x sendet der Sidecar automatisch mTLS, wenn das Ziel einen Sidecar hat (Auto-mTLS). Eine explizite DestinationRule ist nur nötig, wenn man Auto-mTLS überschreiben will.

```bash
kubectl -n bookinfo delete destinationrule ratings-mtls
```

---

## Schritt 8: mTLS-Traffic mit tcpdump verifizieren (Bonus)

**Terminal 1** – Listener auf dem ratings-Pod:

```bash
kubectl -n bookinfo debug -it \
  $(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
  --image=nicolaka/netshoot -- tcpdump -i eth0 -A -c 20 port 9080
```

**Terminal 2** – Request auslösen:

```bash
kubectl -n bookinfo exec deploy/sleep -- curl -s ratings:9080/ratings/0
```

**Frage:** Sehen Sie Klartext oder verschlüsselte Daten im tcpdump-Output?

---

## Aufräumen

```bash
kubectl -n bookinfo delete pod sleep
kubectl -n bookinfo delete sa sleep 
kubectl delete ns no-mesh
```

---

## Zusammenfassung

| Konzept | Beschreibung |
|---------|-------------|
| **PeerAuthentication** | Server-Seite: akzeptiert mTLS (STRICT) oder beides (PERMISSIVE) |
| **DestinationRule** | Client-Seite: sendet mTLS (ISTIO_MUTUAL) oder Klartext (DISABLE) |
| **Auto-mTLS** | Istio aktiviert mTLS automatisch, wenn beide Seiten im Mesh sind |
| **PERMISSIVE** | Default – erlaubt Migration ohne Downtime |
| **STRICT** | Zero-Trust – nur mTLS-Traffic erlaubt |
| **Policy-Hierarchie** | Mesh-weit → Namespace → Workload (spezifischste gewinnt) |

---

## Lösungen

<details>
<summary>Antworten zu den Fragen</summary>

**Schritt 2a:** Istio zeigt standardmäßig `PERMISSIVE` an – Plaintext und mTLS werden akzeptiert.

**Schritt 2c:** Der Header `X-Forwarded-Client-Cert` enthält die SPIFFE-ID des Callers (z.B. `spiffe://cluster.local/ns/bookinfo/sa/sleep`). Das beweist, dass mTLS aktiv ist.

**Schritt 4:** PERMISSIVE ist der Default, weil es eine schrittweise Migration erlaubt – Services können nach und nach ins Mesh aufgenommen werden, ohne dass bestehende Verbindungen brechen. STRICT ist besser für Produktionsumgebungen mit Zero-Trust-Anforderungen.

**Schritt 5:** Istio wertet PeerAuthentication in dieser Reihenfolge aus: Workload-spezifisch (mit `selector`) > Namespace-weit (ohne `selector`, im App-Namespace) > Mesh-weit (ohne `selector`, in `istio-system`). Die spezifischste Policy gewinnt.

**Schritt 6:** Port-Level-Konfiguration ist sinnvoll, wenn ein Service Health-Check-Endpoints hat, die von außerhalb des Mesh (z.B. von einem Load Balancer ohne Sidecar) erreichbar sein müssen.

**Schritt 8:** Bei aktivem mTLS sehen Sie im tcpdump nur verschlüsselte TLS-Daten, keine Klartext-HTTP-Header oder -Bodies.

</details>
