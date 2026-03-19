# Istio Observability: Engpässe finden mit Prometheus & Grafana

## Was ist Latenz?

Die Zeit, die ein Request von Absenden bis Antwort braucht — gemessen in Millisekunden. In Istio misst der **Envoy Sidecar** diese Zeit automatisch für jeden Request.

### Inbound vs. Outbound Latenz

Stell dir Service A → Service B → Service C vor.

**Aus Sicht von Service B:**

- **Inbound-Latenz** = wie lange B insgesamt braucht, um auf einen eingehenden Request zu antworten (gemessen vom Envoy-Proxy *vor* B). Das ist die Gesamtzeit aus Sicht des Aufrufers.
- **Outbound-Latenz** = wie lange die Requests dauern, die B *selbst nach außen* schickt (z.B. an Service C). Gemessen vom Envoy-Proxy *von B* Richtung C.

### Warum das beim Troubleshooting hilft

```
Inbound-Latenz von B:  800ms
Outbound-Latenz von B → C:  750ms
```

→ B selbst ist nicht das Problem — er wartet fast die ganze Zeit auf C.

```
Inbound-Latenz von B:  800ms
Outbound-Latenz von B → C:  50ms
```

→ B selbst braucht ~750ms für seine eigene Verarbeitung — **B ist der Flaschenhals**.

Die Differenz zwischen Inbound und Outbound zeigt dir also, ob der Service selbst langsam ist oder ob er auf einen anderen wartet.

---

## Goldene Signale in Prometheus

### Latenz pro Service (P99)

```promql
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
  by (destination_service, le)
)
```

### Error Rate

```promql
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
by (destination_service)
/
sum(rate(istio_requests_total[5m]))
by (destination_service)
```

### Throughput / Request Rate

```promql
sum(rate(istio_requests_total[5m])) by (destination_service, source_workload)
```

---

## Grafana Dashboards

Istio bringt fertige Dashboards mit — die wichtigsten für Troubleshooting:

- **Istio Mesh Dashboard** → Überblick über das gesamte Mesh (Global Request Rate, Error Rate)
- **Istio Service Dashboard** → Latenz-Heatmap pro Service, P50/P90/P99 auf einen Blick
- **Istio Workload Dashboard** → Inbound/Outbound Latenz pro Pod, hier siehst du, *welcher* Workload bremst

---

## Systematisch den Engpass eingrenzen

1. **Mesh Dashboard** → welcher Service hat hohe Latenz oder Error Rate?
2. **Service Dashboard** → auf den verdächtigen Service klicken → Inbound vs. Outbound Latenz vergleichen
3. **Workload Dashboard** → einzelne Pods prüfen, ob ein bestimmter Pod langsam ist

**Kernlogik:** Wenn die Inbound-Latenz eines Service hoch ist, aber seine Outbound-Latenz zu Downstream-Services niedrig → der Service selbst ist das Problem. Wenn die Outbound-Latenz hoch ist → das Problem liegt weiter downstream.

---

## Zusätzlich: Kiali + Tracing

- **Kiali** zeigt den Service Graph mit farbcodierten Edges (rot = Fehler, gelb = langsam)
- **Jaeger/Zipkin** für Distributed Tracing → einzelne Requests aufschlüsseln, um zu sehen, *wo genau* im Call-Chain die Zeit verbraucht wird

---

## Nützlicher Alert (PrometheusRule)

```yaml
- alert: HighP99Latency
  expr: |
    histogram_quantile(0.99,
      sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
      by (destination_service, le)
    ) > 500
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "P99 Latenz > 500ms für {{ $labels.destination_service }}"
```

---

## Kurzfassung

Mesh Dashboard → auffälligen Service identifizieren → Service/Workload Dashboard → Inbound vs. Outbound Latenz vergleichen → mit Tracing den konkreten Call pinpointen.
