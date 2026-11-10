# Uebung: Saeule 1 - mTLS im Ambient-Mesh pruefen und aktivieren/deaktivieren

> Das ist die **Ambient**-Fassung der Uebung. Die Fassung fuer Sidecars steht in
> [02-check-mtls-enable-disable.md](02-check-mtls-enable-disable.md).

## Ziel

Du pruefst, ob der Verkehr in Bookinfo mit mTLS laeuft, und steuerst mit `PeerAuthentication`, ob Clients **ausserhalb** des Mesh noch durchkommen.
Dabei lernst du, dass im Ambient-Mode der **ztunnel** das mTLS macht und nicht mehr ein Sidecar im Pod.

## Hintergrund: Was ist im Ambient-Mode anders?

| | Sidecar | Ambient |
|---|---|---|
| Wer macht mTLS? | Envoy-Sidecar in jedem Pod | **ztunnel** (ein Pod pro Node) |
| Tunnel | mTLS direkt zwischen den Sidecars | **HBONE** (mTLS) ueber Port **15008** |
| Identitaet | SPIFFE-ID pro ServiceAccount | dieselbe SPIFFE-ID pro ServiceAccount |
| Status pruefen | `istioctl x describe pod` | `istioctl ztunnel-config` und ztunnel-Log |
| Pod hat | 2 Container (App + `istio-proxy`) | 1 Container (nur die App) |
| Client ohne Mesh | wird mit STRICT abgelehnt | wird mit STRICT abgelehnt |

Merksatz: **Zwischen Pods im Mesh laeuft immer mTLS.** `PeerAuthentication` entscheidet nur, ob zusaetzlich noch Klartext von Clients **ausserhalb** des Mesh angenommen wird (PERMISSIVE) oder nicht (STRICT).

## Voraussetzungen

  * Uebung [Bookinfo demo in ambient ausrollen](../../installation/ambient/04-install-demo-app.md) ist durchgefuehrt
    (Namespace `bookinfo` mit Waypoint, alle Pods `1/1 Running`).
  * `istioctl` ist installiert.
  * Im Namespace gibt es keine PeerAuthentication und keine AuthorizationPolicy:

```
kubectl get peerauthentication -A
kubectl -n bookinfo get authorizationpolicy
```

Erwartet: `No resources found`. Sonst vorher loeschen, z. B. `kubectl -n bookinfo delete peerauthentication --all`

Der Namespace muss im Ambient-Mode sein (kein `istio-injection`!):

```
kubectl get ns bookinfo --show-labels
```

Erwartet: `istio.io/dataplane-mode=ambient` (und `istio.io/use-waypoint=waypoint`)

---

## Schritt 1: Vorbereitung und Funktionstest

Ein kleiner Client im Mesh:

```
kubectl -n bookinfo apply -f ~/istio/samples/sleep/sleep.yaml
kubectl -n bookinfo rollout status deploy/sleep
```

```
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

Erwartete Ausgabe: `<title>Simple Bookstore App</title>`

Die Pods haben nur **einen** Container, es gibt keinen Sidecar:

```
kubectl -n bookinfo get pods
```

---

## Schritt 2: Aktuellen mTLS-Status pruefen

### 2a) Warum `istioctl x describe` hier nicht hilft

In der Sidecar-Uebung haben wir den Status mit `istioctl x describe pod` gelesen. Probiere es hier aus:

```
istioctl x describe pod \
  $(kubectl -n bookinfo get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  -n bookinfo
```

Du siehst eine Meldung wie:

```
WARNING: productpage-v1-... is not part of mesh; no Istio sidecar
Error: failed to execute command on sidecar: ...
```

**Frage:** Ist der Pod wirklich nicht im Mesh? Woher kommt die Meldung?

`describe` fragt den Envoy-Sidecar im Pod ab (Port 15000). Im Ambient-Mode gibt es den nicht. Die Meldung "not part of mesh" ist deshalb **irrefuehrend**: Der Pod ist im Mesh, nur ohne Sidecar.

### 2b) Wird der Pod vom ztunnel bedient? (`ztunnel-config workload`)

```
istioctl ztunnel-config workload | grep -E "NAMESPACE|bookinfo"
```

Du siehst in der Spalte `PROTOCOL` fuer die Bookinfo-Pods `HBONE` und in der Spalte `WAYPOINT` den Namen `waypoint`. Der Waypoint selbst und das Gateway stehen mit `TCP` da.

**Frage:** Was bedeutet `HBONE`, was bedeutet `TCP`?

### 2c) Welche Zertifikate (SPIFFE-IDs) hat der ztunnel?

`ztunnel-config certificate` braucht den Namen des ztunnel-Pods. Wir nehmen den ztunnel auf dem Node, auf dem `ratings` laeuft:

```
NODE=$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].spec.nodeName}')
ZT=$(kubectl -n istio-system get pod -l app=ztunnel --field-selector spec.nodeName=$NODE -o jsonpath='{.items[0].metadata.name}')
echo $ZT
```

```
istioctl ztunnel-config certificate $ZT.istio-system | grep -E "CERTIFICATE|bookinfo"
```

Du siehst je ServiceAccount ein `Leaf`-Zertifikat (kurz gueltig) und das gemeinsame `Root`-Zertifikat, z. B.:

```
spiffe://cluster.local/ns/bookinfo/sa/bookinfo-ratings   Leaf   Available   true   ...
```

**Frage:** Warum stehen hier nur die ServiceAccounts, deren Pods auf diesem Node laufen?

### 2d) Bestehende PeerAuthentication-Policies anzeigen

```
# Ueberall im Cluster
kubectl get peerauthentication -A

# Im Namespace
kubectl -n bookinfo get peerauthentication
```

> **Hinweis:** `PeerAuthentication` ist eine **Namespaced Resource**. Die Geltung haengt davon ab, *wo* sie liegt:
>
> - Im **App-Namespace** (z. B. `bookinfo`) gilt sie fuer diesen Namespace (ohne `selector`) oder fuer einzelne Workloads (mit `selector`).
> - In **`istio-system`** wirkt sie als Mesh-weiter Default.
>
> Ist nirgends eine PeerAuthentication gesetzt (`No resources found`), nimmt der ztunnel auch Klartext von Clients ausserhalb des Mesh an. Das pruefen wir in Schritt 3.

### 2e) Verifizierung ueber das ztunnel-Log

Im Sidecar-Mode haben wir den Header `X-Forwarded-Client-Cert` angeschaut. Im Ambient-Mode steht die Identitaet im **ztunnel-Log**.

Zuerst einen Aufruf ausloesen:

```
kubectl -n bookinfo exec deploy/sleep -- curl -s -o /dev/null ratings:9080/ratings/0
```

Dann den ztunnel auf dem Node des **Aufrufers** (`sleep`) anschauen:

```
NODE_SLEEP=$(kubectl -n bookinfo get pod -l app=sleep -o jsonpath='{.items[0].spec.nodeName}')
ZT_SLEEP=$(kubectl -n istio-system get pod -l app=ztunnel --field-selector spec.nodeName=$NODE_SLEEP -o jsonpath='{.items[0].metadata.name}')
kubectl -n istio-system logs $ZT_SLEEP --tail=100 | grep 'src.workload="sleep' | tail -1
```

Die Zeile sieht gekuerzt so aus:

```
info access connection complete src.workload="sleep-..." src.namespace="bookinfo"
  src.identity="spiffe://cluster.local/ns/bookinfo/sa/sleep"
  dst.addr=192.168.x.x:15008 dst.hbone_addr=10.x.x.x:9080 dst.service="ratings.bookinfo.svc.cluster.local"
  dst.workload="waypoint-..." dst.identity="spiffe://cluster.local/ns/bookinfo/sa/waypoint" ...
```

**Fragen:**
  * Welche SPIFFE-ID hat der Aufrufer (`src.identity`)?
  * Auf welchem Port geht die Verbindung raus (`dst.addr`)? Was ist das fuer ein Port?
  * Warum ist das Ziel (`dst.workload`) der **Waypoint** und nicht `ratings`?

---

## Schritt 3: mTLS auf STRICT setzen

### 3a) Vorher: Client ohne Mesh

Ein Pod in einem Namespace **ohne** Ambient-Label ist nicht im Mesh:

```
kubectl create ns no-mesh
kubectl -n no-mesh apply -f ~/istio/samples/sleep/sleep.yaml
kubectl -n no-mesh rollout status deploy/sleep
```

```
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage

kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0
```

Erwartet: zweimal `200`. Ohne Policy kommt auch Klartext von ausserhalb durch.

### 3b) Namespace-weite STRICT Policy

```
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

### 3c) Verifizieren - die Kommunikation im Mesh funktioniert weiterhin

```
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage:9080/productpage

kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

Erwartet: zweimal `200`.

### 3d) Zugriff ohne Mesh testen

```
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage

kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0
```

**Erwartetes Ergebnis:** Beide Aufrufe schlagen fehl. curl gibt `000` aus (keine HTTP-Antwort) und bricht mit einem Fehler ab (Exit-Code 52 oder 56, je nachdem, wie die Verbindung geschlossen wird). Der ztunnel des Ziel-Pods schliesst die Verbindung, bevor es zu HTTP kommt.

### 3e) Warum? Der Grund steht im ztunnel-Log des Ziels

```
NODE=$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].spec.nodeName}')
ZT=$(kubectl -n istio-system get pod -l app=ztunnel --field-selector spec.nodeName=$NODE -o jsonpath='{.items[0].metadata.name}')
kubectl -n istio-system logs $ZT --tail=50 | grep 'src.namespace="no-mesh"' | tail -1
```

Am Ende der Zeile steht:

```
direction="inbound" ... error="connection closed due to policy rejection: explicitly denied by: istio-system/istio_converted_static_strict"
```

**Frage:** Welche Seite lehnt ab, die Quelle oder das Ziel? Woran siehst du das (`direction`)?

---

## Schritt 4: mTLS auf PERMISSIVE setzen

```
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

```
# Aus dem Mesh - funktioniert (mit mTLS)
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage:9080/productpage

# Von ausserhalb des Mesh - funktioniert jetzt wieder (ohne mTLS)
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage
```

**Frage:** Ist der Aufruf aus dem Mesh bei PERMISSIVE verschluesselt? Pruefe es mit dem ztunnel-Log aus Schritt 2e (`dst.addr` Port 15008).

---

## Schritt 5: Workload-spezifisches mTLS

Wir setzen nur `ratings` auf STRICT, der Rest bleibt PERMISSIVE:

```
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

```
# ratings ist jetzt STRICT - von ausserhalb nicht erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0

# productpage ist weiterhin PERMISSIVE - von ausserhalb erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage

# Aus dem Mesh kommt ratings weiterhin an
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

**Frage:** In welcher Reihenfolge wertet Istio PeerAuthentication-Policies aus?

> **Wichtig:** Lass `ratings-strict` **stehen**, wir brauchen die Policy gleich in Schritt 6.

---

## Schritt 6: Port-Level-mTLS (Bonus)

mTLS laesst sich auch pro Port konfigurieren:

```
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

Die Policy `ratings-strict` aus Schritt 5 ist noch da. Warte ca. 10 Sekunden und teste von ausserhalb des Mesh:

```
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0
```

**Erwartet: der Aufruf schlaegt fehl.** curl gibt `000` aus (Exit-Code 52 oder 56), obwohl Port 9080 in `ratings-per-port` auf PERMISSIVE steht.

**Frage:** Warum greift die Ausnahme fuer Port 9080 nicht? (Tipp: Schau dir die Policies sortiert nach Alter an.)

```
kubectl -n bookinfo get peerauthentication --sort-by=.metadata.creationTimestamp
```

Jetzt loeschen wir die aeltere Policy `ratings-strict`, warten wieder ca. 10 Sekunden und testen noch einmal:

```
kubectl -n bookinfo delete peerauthentication ratings-strict
```

```
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0
```

Erwartet: `200`, weil jetzt nur noch `ratings-per-port` gilt und Port 9080 dort auf PERMISSIVE steht, obwohl der Workload sonst STRICT ist.

> **Info: Mehrere PeerAuthentications auf denselben Workload**
>
> Passen mehrere workload-spezifische PeerAuthentications (mit `selector`) auf denselben Workload, gilt nur **eine**: die **aelteste**. Die anderen werden ignoriert, sie werden nicht zusammengefuehrt. Eine aeltere, reine STRICT-Policy ueberstimmt deshalb eine juengere mit Port-Ausnahme, unabhaengig davon, welche "strenger" ist.
>
> Aus der Istio-Doku (Abschnitt "Selector field"):
>
> - "When more than one workload-specific peer authentication policy matches, Istio picks the oldest one."
> - "When you configure multiple mesh- or namespace-wide peer authentication policies for the same mesh or namespace, Istio ignores the newer policies."
>
> Quelle: <https://istio.io/latest/docs/concepts/security/#selector-field>
>
> Zu `portLevelMtls` sagt die [PeerAuthentication-Referenz](https://istio.io/latest/docs/reference/config/security/peer_authentication/): "The port refers to the port of the workload, not the port of the Kubernetes service." Eingetragen wird also der Port des Containers (`targetPort`), nicht der des Service. Bei `ratings` sind beide `9080`.
>
> Praxis: Pro Workload nur **eine** PeerAuthentication mit `selector` anlegen und die Port-Ausnahme dort mit `portLevelMtls` setzen.

**Frage:** Fuer welches Szenario ist Port-Level-Konfiguration sinnvoll?

```
kubectl -n bookinfo delete peerauthentication ratings-per-port
```

---

## Schritt 7: Client-Seite - was aendert sich im Ambient-Mode?

mTLS auf der Client-Seite passiert in **beiden** Modi automatisch (Auto-mTLS): Im Sidecar-Mode sendet der Sidecar des Clients mTLS, sobald das Ziel einen Sidecar hat. Im Ambient-Mode baut der ztunnel der Quelle zu jedem Ziel im Mesh den HBONE-Tunnel auf.

Der Unterschied ist der **Hebel zum Ueberschreiben**:

| | Sidecar | Ambient |
|---|---|---|
| mTLS zum Ziel | automatisch (Auto-mTLS) | automatisch (HBONE durch den ztunnel) |
| Ueberschreiben per `DestinationRule` (`tls.mode: DISABLE` oder `ISTIO_MUTUAL`) | moeglich, der Sidecar des Clients richtet sich danach | **wirkungslos**, mTLS ist fest |

Du kannst es nachstellen: `ratings` auf STRICT setzen und dazu eine `DestinationRule` mit `tls.mode: DISABLE` anlegen:

```
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

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ratings-disable
  namespace: bookinfo
spec:
  host: ratings.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
EOF
```

```
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

Erwartet: `200`. Der Aufruf kommt trotz `DISABLE` und STRICT durch und laeuft weiter ueber Port 15008 (ztunnel-Log wie in Schritt 2e). Im Sidecar-Mode wuerde `DISABLE` Klartext senden, und STRICT wuerde ihn ablehnen.

```
kubectl -n bookinfo delete destinationrule ratings-disable
kubectl -n bookinfo delete peerauthentication ratings-strict
```

**Frage:** Wo wuerdest du im Ambient-Mode eine `DestinationRule` trotzdem noch brauchen? (Tipp: Waypoint, Layer 7)

---

## Aufraeumen

```
kubectl -n bookinfo delete peerauthentication --all
kubectl -n bookinfo delete deploy sleep
kubectl -n bookinfo delete svc sleep
kubectl -n bookinfo delete sa sleep
kubectl delete ns no-mesh
```

---

## Zusammenfassung

| Konzept | Beschreibung |
|---------|-------------|
| **ztunnel** | Macht mTLS fuer alle Pods auf seinem Node (HBONE, Port 15008) |
| **Status pruefen** | `istioctl ztunnel-config workload` (`HBONE`/`TCP`), `ztunnel-config certificate`, ztunnel-Log |
| **`x describe`** | Nur fuer Sidecars, im Ambient-Mode nutzlos und irrefuehrend |
| **PeerAuthentication** | Entscheidet, ob Klartext von Clients ausserhalb des Mesh angenommen wird (PERMISSIVE) oder nicht (STRICT) |
| **Ohne Policy** | Clients ausserhalb des Mesh kommen durch (wie PERMISSIVE) |
| **STRICT** | Zero-Trust: nur mTLS, der ztunnel des Ziels lehnt Klartext ab |
| **Client-Seite** | mTLS ist automatisch (wie bei Sidecars, dort Auto-mTLS). Neu im Ambient-Mode: Der ztunnel der Quelle baut den Tunnel, `tls.mode` in einer `DestinationRule` ist wirkungslos |
| **Policy-Hierarchie** | Mesh-weit (`istio-system`) -> Namespace -> Workload (die spezifischste gewinnt) |

---

## Loesungen

<details>
<summary>Antworten zu den Fragen</summary>

**Schritt 2a:** Der Pod ist im Mesh. `istioctl x describe` sucht aber den Envoy-Sidecar im Pod und findet keinen, weil im Ambient-Mode der ztunnel auf dem Node arbeitet. Deshalb die irrefuehrende Meldung "not part of mesh; no Istio sidecar".

**Schritt 2b:** `HBONE` heisst: Der Verkehr zu diesem Pod laeuft ueber den mTLS-Tunnel des ztunnel (Port 15008). `TCP` heisst: Der Pod ist nicht im Ambient-Mesh (hier Waypoint und Gateway), der Verkehr geht direkt und ohne ztunnel-mTLS.

**Schritt 2c:** Jeder ztunnel holt Zertifikate nur fuer die ServiceAccounts, deren Pods auf seinem Node laufen. Fuer andere Identitaeten fragt er nicht.

**Schritt 2e:** `src.identity` ist `spiffe://cluster.local/ns/bookinfo/sa/sleep` (Namespace und ServiceAccount des Aufrufers). Der Port ist 15008, der HBONE-Port: Der Verkehr ist mTLS-verschluesselt. Das Ziel ist der Waypoint, weil der Namespace `istio.io/use-waypoint=waypoint` hat: Der ztunnel schickt den Verkehr zuerst zum Waypoint (Layer 7), der danach `ratings` aufruft.

**Schritt 3e:** Das Ziel lehnt ab: `direction="inbound"` steht fuer die eingehende Seite des ztunnel auf dem Node von `ratings`. Die Policy `istio_converted_static_strict` ist die STRICT-PeerAuthentication, die der ztunnel intern in eine Regel umwandelt.

**Schritt 4:** Ja. Die Verbindung zwischen Pods im Mesh nutzt weiterhin HBONE (Port 15008) und mTLS. PERMISSIVE erlaubt nur zusaetzlich Klartext von Clients ausserhalb des Mesh.

**Schritt 5:** Istio wertet PeerAuthentication in dieser Reihenfolge aus: Workload-spezifisch (mit `selector`) > Namespace-weit (ohne `selector`, im App-Namespace) > Mesh-weit (ohne `selector`, in `istio-system`). Die spezifischste Policy gewinnt.

**Schritt 6 (Warum geht es zuerst nicht?):** Auf `ratings` passen jetzt zwei workload-spezifische PeerAuthentications: `ratings-strict` (Schritt 5) und `ratings-per-port`. Istio nimmt davon nur die aelteste, und das ist `ratings-strict`, ein reines STRICT ohne Port-Ausnahme. `ratings-per-port` wird ignoriert, deshalb wird Klartext von ausserhalb weiter abgelehnt. Nach dem Loeschen von `ratings-strict` gilt `ratings-per-port`, und Port 9080 ist PERMISSIVE. Die Reihenfolge beim Anlegen entscheidet, nicht die Strenge.

**Schritt 6 (Szenario):** Port-Level-Konfiguration ist sinnvoll, wenn ein Port (z. B. ein Health-Check- oder Metrics-Endpunkt) auch von Clients ausserhalb des Mesh erreichbar sein muss, waehrend der Rest des Workloads STRICT bleibt.

**Schritt 7:** Fuer Layer-7-Verhalten, das der Waypoint umsetzt, z. B. Subsets, Load-Balancing (`LEAST_CONN`) oder Circuit Breaker. Die mTLS-Einstellung dagegen kannst du per `DestinationRule` nicht mehr beeinflussen: mTLS ist im Ambient-Mode automatisch und fest (`tls.mode: DISABLE` wird ignoriert, siehe Test oben).

</details>
