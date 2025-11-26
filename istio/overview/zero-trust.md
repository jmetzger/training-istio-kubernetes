# 🔐 Zero-Trust in Istio – die 5 zentralen Prinzipien

## 1️⃣ **Strong Authentication (mTLS überall)**

* Jeder Pod bekommt automatisch ein **X.509-Zertifikat** (~SpiffeID).
* **Alle** Services sprechen untereinander **mutual TLS**.
* Istio überprüft:

  * Ist der Client wirklich der, der er vorgibt zu sein?
  * Passt die SPIFFE Identity?

👉 *Nichts darf unverschlüsselt, nichts darf anonym miteinander reden.*

---

## 2️⃣ **No implicit trust**

Mit einer deny - policy (oder einer einzigen Allow-Policy) gilt heute in Istio:
**Alles ist DENY, bis du ALLOW definierst.**

D. h. nur weil zwei Services im gleichen Namespace laufen, dürfen sie sich **nicht automatisch** gegenseitig aufrufen.

→ Du definierst explizit **RequestAuthentication** und **AuthorizationPolicy**.

---

## 3️⃣ **Fine-grained Authorization**

Istio entscheidet:

* **Wer** (Service identity / JWT claims)
* **darf** (ALLOW)
* **was** (HTTP-Verb, Pfad, Port)
* **wohin** (Service, Namespace)
* **von wo** (IP, Namespace, Principals)

Beispiel:
„Nur `reviews` darf `/ratings/*` aufrufen – aber nur GET, nicht POST.“

Das ist Zero-Trust.

---

## 4️⃣ **Policy enforcement auf Service-Ebene**

Alle Regeln gelten **zentral**, unabhängig vom Code des Services.
Das heißt:

* Keine ACLs mehr im Code
* Keine Bibliotheken anpassen
* Keine Firewall-Regeln auf Node-Ebene

→ Der Sidecar (oder Waypoint) erzwingt die Security.

---

## 5️⃣ **Identity-based Security (nicht IP-basiert)**

In Kubernetes ändern sich IPs permanent → unbrauchbar.

Istio arbeitet stattdessen mit **Identitäten**, z. B.:

```
spiffe://cluster.local/ns/bookinfo/sa/productpage
```

→ Diese Identität wird über mTLS geprüft

---

# 🧠 Merksatz

**Zero-Trust in Istio:
„Authenticate everything, authorize explicitly, trust nobody automatically.“**


* als **1-Folie** für dein Helm/Istio-Training
* als **Übung** mit echten Policies (ALLOW → DENY Prinzip)
