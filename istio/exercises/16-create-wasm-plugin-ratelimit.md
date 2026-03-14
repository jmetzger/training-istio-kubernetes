# Lab: Istio WASM Rate Limiter — Token Bucket (Go)

## Lernziele

- WASM-Plugin in Go für Envoy/Istio entwickeln
- Token-Bucket-Algorithmus implementieren
- WasmPlugin-Resource konfigurieren (Sidecar, Ambient, Gateway)

## Voraussetzungen

- Kubernetes-Cluster mit Istio ≥ 1.22
- Container-Registry (Harbor, GHCR, Docker Hub)

---

## 1 — Tooling installieren

```bash
# Go (https://go.dev/dl/)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# TinyGo — WASM-fähiger Go-Compiler (Standard-Go kann kein wasi-Target)
wget https://github.com/tinygo-org/tinygo/releases/download/v0.33.0/tinygo_0.33.0_amd64.deb
sudo dpkg -i tinygo_0.33.0_amd64.deb

# ORAS — OCI-Registry-Tool zum Pushen von WASM-Modulen
curl -LO https://github.com/oras-project/oras/releases/download/v1.2.0/oras_1.2.0_linux_amd64.tar.gz
tar xzf oras_1.2.0_linux_amd64.tar.gz && sudo mv oras /usr/local/bin/

# Prüfen
go version && tinygo version && oras version
```

---

## 2 — Projekt anlegen

```bash
mkdir -p istio-ratelimit-wasm && cd istio-ratelimit-wasm
go mod init github.com/t3company/istio-ratelimit-wasm
go get github.com/tetratelabs/proxy-wasm-go-sdk
```

---

## 3 — Token Bucket Algorithmus

```
 Tokens: ████████░░  (8/10)

   Request → Token verbrauchen → 7/10 → 200 OK
   Request → Token verbrauchen → 6/10 → 200 OK
   ...
   Request → Bucket leer       → 0/10 → 429!

   Timer Tick (1s) → Refill +10 → 10/10
```

| Parameter | Beschreibung |
|-----------|-------------|
| `max_tokens` | Burst-Kapazität |
| `refill_rate` | Tokens/Sekunde nachgefüllt |

Vorteil gegenüber Fixed Window: kein Boundary-Problem, Burst-fähig, glättet langfristigen Durchsatz.

---

## 4 — Plugin implementieren

### Proxy-WASM Architektur (Go)

```
VMContext → PluginContext → HttpContext
                │                │
                ├─ OnPluginStart()          Config + Timer
                ├─ OnTick()                 Token Refill
                └─ NewHttpContext()
                         └─ OnHttpRequestHeaders()  Token prüfen
```

### main.go

```go
package main

import (
	"encoding/binary"
	"encoding/json"
	"fmt"

	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

func main() { proxywasm.SetVMContext(&vmContext{}) }

// ── Config ──────────────────────────────────────────────────────────────────

type rateLimitConfig struct {
	MaxTokens        uint64 `json:"max_tokens"`
	RefillRate       uint64 `json:"refill_rate"`
	RefillIntervalMs uint64 `json:"refill_interval_ms"`
	StatusCode       uint32 `json:"status_code"`
	ResponseBody     string `json:"response_body"`
}

func defaultConfig() rateLimitConfig {
	return rateLimitConfig{100, 10, 1000, 429, "rate limit exceeded"}
}

// ── Shared Data Keys + Helpers ──────────────────────────────────────────────

const (
	bucketKey       = "ratelimit_tokens"
	statsAllowedKey = "ratelimit_allowed"
	statsDeniedKey  = "ratelimit_denied"
)

func toBytes(v uint64) []byte {
	b := make([]byte, 8)
	binary.LittleEndian.PutUint64(b, v)
	return b
}

func fromBytes(b []byte) uint64 {
	if len(b) < 8 { return 0 }
	return binary.LittleEndian.Uint64(b)
}

func incrementStat(key string) {
	data, cas, err := proxywasm.GetSharedData(key)
	if err != nil { return }
	proxywasm.SetSharedData(key, toBytes(fromBytes(data)+1), cas)
}

// ── VM Context ──────────────────────────────────────────────────────────────

type vmContext struct{ types.DefaultVMContext }

func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
	return &pluginContext{config: defaultConfig()}
}

// ── Plugin Context (Config + Timer + Bucket) ────────────────────────────────

type pluginContext struct {
	types.DefaultPluginContext
	config rateLimitConfig
}

func (p *pluginContext) OnPluginStart(configSize int) types.OnPluginStartStatus {
	if configSize > 0 {
		if data, err := proxywasm.GetPluginConfiguration(); err == nil {
			var cfg rateLimitConfig
			if err := json.Unmarshal(data, &cfg); err == nil {
				if cfg.MaxTokens == 0        { cfg.MaxTokens = 100 }
				if cfg.RefillRate == 0       { cfg.RefillRate = 10 }
				if cfg.RefillIntervalMs == 0 { cfg.RefillIntervalMs = 1000 }
				if cfg.StatusCode == 0       { cfg.StatusCode = 429 }
				if cfg.ResponseBody == ""    { cfg.ResponseBody = "rate limit exceeded" }
				p.config = cfg
			}
		}
	}

	proxywasm.LogInfof("Rate Limiter: max_tokens=%d, refill_rate=%d/s",
		p.config.MaxTokens, p.config.RefillRate)

	proxywasm.SetSharedData(bucketKey, toBytes(p.config.MaxTokens), 0)
	proxywasm.SetSharedData(statsAllowedKey, toBytes(0), 0)
	proxywasm.SetSharedData(statsDeniedKey, toBytes(0), 0)
	proxywasm.SetTickPeriodMilliSeconds(uint32(p.config.RefillIntervalMs))
	return types.OnPluginStartStatusOK
}

func (p *pluginContext) OnTick() {
	data, cas, err := proxywasm.GetSharedData(bucketKey)
	if err != nil { return }

	current := fromBytes(data)
	add := p.config.RefillRate * p.config.RefillIntervalMs / 1000
	newTokens := current + add
	if newTokens > p.config.MaxTokens { newTokens = p.config.MaxTokens }

	if err := proxywasm.SetSharedData(bucketKey, toBytes(newTokens), cas); err != nil {
		proxywasm.LogWarn("refill CAS conflict, retry next tick")
	}
}

func (p *pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
	return &httpContext{config: p.config}
}

// ── HTTP Context (pro Request) ──────────────────────────────────────────────

type httpContext struct {
	types.DefaultHttpContext
	config rateLimitConfig
}

func (h *httpContext) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
	for attempt := 0; attempt < 3; attempt++ {
		data, cas, err := proxywasm.GetSharedData(bucketKey)
		if err != nil { continue }

		current := fromBytes(data)

		if current == 0 {
			incrementStat(statsDeniedKey)
			proxywasm.SendHttpResponse(h.config.StatusCode, [][2]string{
				{"x-ratelimit-limit", fmt.Sprintf("%d", h.config.MaxTokens)},
				{"x-ratelimit-remaining", "0"},
				{"retry-after", "1"},
			}, []byte(h.config.ResponseBody), -1)
			return types.ActionPause
		}

		if err := proxywasm.SetSharedData(bucketKey, toBytes(current-1), cas); err != nil {
			continue // CAS-Konflikt → retry
		}

		incrementStat(statsAllowedKey)
		proxywasm.AddHttpRequestHeader("x-ratelimit-remaining", fmt.Sprintf("%d", current-1))
		return types.ActionContinue
	}

	proxywasm.LogWarn("CAS exhausted, fail-open")
	return types.ActionContinue
}
```

### Wichtige Konzepte

- **CAS (Compare-And-Swap):** `GetSharedData` liefert Wert + CAS-Token. `SetSharedData` schreibt nur wenn Token noch gültig. Bei Konflikt → retry.
- **Fail-Open:** Nach 3 CAS-Retries wird der Request durchgelassen — kaputtes Plugin soll nicht allen Traffic blockieren.
- **`types.DefaultHttpContext`:** Go-Embedding, implementiert alle Interface-Methoden als No-Ops. Wir überschreiben nur was wir brauchen.

---

## 5 — Build & Push

```bash
# Kompilieren (-scheduler=none: kein Goroutine-Scheduler, -target=wasi: WASM)
tinygo build -o ratelimit.wasm -scheduler=none -target=wasi main.go

ls -lh ratelimit.wasm   # ~500 KB - 1.5 MB

# Optional: ~30% kleiner
wasm-opt -Os ratelimit.wasm -o ratelimit.wasm

# Pushen
oras login registry.example.com
oras push registry.example.com/wasm/ratelimit:v1 \
  ratelimit.wasm:application/vnd.module.wasm.content.layer.v1+wasm
```

---

## 6 — Deployen

```bash
kubectl create namespace demo
kubectl label namespace demo istio-injection=enabled
kubectl apply -f k8s/test-deployment.yaml
kubectl -n demo get pods -w   # Warten auf 2/2 Running
```

`k8s/test-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: demo
spec:
  selector:
    app: my-service
  ports:
    - port: 8080
      targetPort: 80
```

`k8s/wasmplugin.yaml`:

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: local-ratelimit
  namespace: demo
spec:
  selector:
    matchLabels:
      app: my-service
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN                    # Vor Authentication in der Filterkette
  pluginConfig:
    max_tokens: 100
    refill_rate: 10
    refill_interval_ms: 1000
    status_code: 429
    response_body: '{"error": "rate limit exceeded", "retry_after": 1}'
  match:
    - mode: SERVER                # Nur Inbound
      ports:
        - number: 8080
  imagePullPolicy: IfNotPresent
```

```bash
kubectl apply -f k8s/wasmplugin.yaml
kubectl -n demo logs deploy/my-service -c istio-proxy | grep -i "rate"
```

---

## 7 — Testen

```bash
# Einzelner Request
kubectl -n demo run test --rm -it --image=curlimages/curl -- \
  curl -sv http://my-service:8080/get 2>&1 | grep -E "HTTP|x-ratelimit"

# Load Test
kubectl -n demo run loadtest --rm -it --image=curlimages/curl -- sh -c '
  ALLOWED=0; DENIED=0
  for i in $(seq 1 200); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://my-service:8080/get)
    [ "$STATUS" = "429" ] && DENIED=$((DENIED + 1)) || ALLOWED=$((ALLOWED + 1))
  done
  echo "Erlaubt: $ALLOWED | Gedrosselt: $DENIED | Gesamt: 200"
'
```

**Erwartet:** ~100 erlaubt (Burst), Rest 429. Pro Sekunde kommen ~10 nach.

---

## 8 — Config ändern

`max_tokens: 10`, `refill_rate: 2` setzen → `kubectl apply` → Load Test wiederholen. Erwartet: nur ~10 erlaubt.

> **Frage:** Was passiert bei 3 Replicas? → Jeder Pod hat eigenen Bucket → 3× Durchsatz. Das ist Local vs. Global Rate Limiting.

---

## 9 — Ambient Mode

```bash
kubectl label namespace demo istio-injection-
kubectl label namespace demo istio.io/dataplane-mode=ambient
istioctl waypoint apply -n demo --name my-service-waypoint --for service
kubectl -n demo label service my-service istio.io/use-waypoint=my-service-waypoint
```

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: local-ratelimit-ambient
  namespace: demo
spec:
  targetRefs:                       # statt selector
    - kind: Service
      group: ""
      name: my-service
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN
  pluginConfig:
    max_tokens: 50
    refill_rate: 5
    refill_interval_ms: 1000
```

Waypoint Proxy = ein zentraler Proxy pro Service → ein Bucket → verhält sich wie Global Rate Limiting.

---

## 10 — Gateway-Level Rate Limiting

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: gateway-ratelimit
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN
  pluginConfig:
    max_tokens: 1000
    refill_rate: 100
```

Gateway schützt den Cluster, Service-Level schützt einzelne Microservices. In Produktion beide kombinieren.

---

## 11 — Bonus-Aufgaben

**Per Source-IP:** `proxywasm.GetHttpRequestHeader("x-forwarded-for")` → Bucket-Key pro IP.

**Health-Checks ausnehmen:** `:path` Header prüfen, bei `/health` direkt `ActionContinue`.

**Per Pfad:** Separate Config pro Pfad-Prefix in `pluginConfig`.

---

## Vergleich

| | EnvoyFilter | WasmPlugin |
|--|-------------|------------|
| API-Stabilität | "break glass" | Stabile Istio-API |
| Portierbarkeit | Istio-only | Envoy, Istio, Gateway API |
| Ambient Mode | Nein | Ja |

| | Go (TinyGo) | Rust |
|--|-------------|------|
| Lernkurve | Flach | Steil |
| Binary-Größe | ~500 KB - 1.5 MB | ~200-400 KB |
| Einschränkungen | Kein `net/http`, kein `reflect` | Keine |
| Empfehlung | Prototyping, Go-Teams | Produktion, Performance |

---

## Aufräumen

```bash
kubectl -n demo delete wasmplugin --all && kubectl delete namespace demo
```

## Referenzen

- [Proxy-WASM Go SDK](https://github.com/tetratelabs/proxy-wasm-go-sdk) · [TinyGo](https://tinygo.org/) · [Istio WasmPlugin API](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/) · [ORAS](https://oras.land/docs/)
