# Wildcards bei `principals` in der AuthorizationPolicy

**Stand: 10.11.2026.** Aus allgemeinem Istio-Wissen zusammengefasst, **nicht auf dem Trainings-Cluster getestet** (ungeprüft).

## Kurzfassung

Ja, aber nur eingeschränkt. Istio kennt bei `principals` **kein freies Globbing**, nur drei Formen:

| Form | Beispiel | Bedeutung |
|---|---|---|
| Präfix | `cluster.local/ns/team-a/sa/*` | alle ServiceAccounts im Namespace `team-a` |
| Suffix | `*/ns/team-a/sa/frontend` | `frontend` in `team-a`, egal welche Trust Domain |
| Presence | `*` | jede authentifizierte Identität (Request mit mTLS) |

Dazu kommt der exakte Treffer ohne Wildcard.

## Aufbau einer Principal

```
<trust-domain>/ns/<namespace>/sa/<serviceaccount>
```

* Standard-Trust-Domain ist `cluster.local`.
* **Kein** `spiffe://` davor schreiben (in der Zertifikats-URI steht es, in der Policy nicht).

Beispiel: `cluster.local/ns/bookinfo/sa/bookinfo-reviews`

## Beispiel: alle ServiceAccounts eines Namespace

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-team-a
  namespace: backend
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/team-a/sa/*"]
```

## Was nicht geht

* **`*` in der Mitte**, zum Beispiel `cluster.local/ns/*/sa/frontend`. Das wird als exakter String behandelt und trifft nie.
* **Präfix und Suffix zugleich**, zum Beispiel `*/ns/team-a/sa/*`. Pro Eintrag ist nur ein `*` am Anfang **oder** am Ende möglich.
* **Reguläre Ausdrücke.**

Wer "frontend in beliebigem Namespace" braucht, muss die Namespaces einzeln auflisten.

## Stolperfallen

* **`principals: ["*"]` heißt nicht "alle".** Plaintext-Traffic ohne Identität trifft nicht. Will man wirklich alle zulassen, lässt man `from` weg.
* Für "alle aus einem Namespace" ist `namespaces: ["team-a"]` meist sauberer als das Präfix in `principals`.
* `notPrincipals` kennt dieselben Wildcard-Formen.
* Dieselben Formen (exakt, Präfix, Suffix, Presence) gelten auch bei anderen Feldern der Policy, etwa `hosts`, `paths`, `methods`. Nicht dagegen bei `ipBlocks`, dort steht CIDR.

## Ambient

Principal-basierte Regeln wertet schon **ztunnel auf Layer 4** aus, ein Waypoint ist dafür nicht nötig. Die Wildcards verhalten sich wie beim Sidecar.

Regeln mit Layer-7-Feldern (`methods`, `paths`, `hosts`) brauchen dagegen einen Waypoint. Mehr dazu in der [Übung Zero Trust Ambient](../exercises/19-zerotrust-bookinfo-ambient.md).

## Weiterlesen

* Istio-Doku: [Authorization Policy, Referenz](https://istio.io/latest/docs/reference/config/security/authorization-policy/) (Abschnitt `Source`)
