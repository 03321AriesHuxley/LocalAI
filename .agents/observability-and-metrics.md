# Observability and Metrics in LocalAI

This guide covers how LocalAI exposes metrics, traces, and structured logs for monitoring and debugging in production environments.

## Overview

LocalAI uses the following observability stack:
- **Prometheus** for metrics collection
- **OpenTelemetry (OTel)** for distributed tracing
- **Structured JSON logging** via `zerolog`

---

## Metrics

### Prometheus Endpoint

Metrics are exposed at `GET /metrics` (default port `8080`).

To enable metrics, set the following in your config or environment:

```bash
LOCALAI_METRICS=true
LOCALAI_METRICS_PORT=8080  # optional, defaults to same as API port
```

### Available Metrics

| Metric Name | Type | Description |
|---|---|---|
| `localai_api_requests_total` | Counter | Total number of API requests by endpoint and status |
| `localai_api_request_duration_seconds` | Histogram | Request latency by endpoint |
| `localai_model_load_duration_seconds` | Histogram | Time taken to load a model into memory |
| `localai_model_inference_duration_seconds` | Histogram | Time taken per inference call |
| `localai_models_loaded` | Gauge | Number of currently loaded models |
| `localai_tokens_generated_total` | Counter | Total tokens generated across all models |
| `localai_backend_errors_total` | Counter | Total backend errors by backend name |

### Registering Custom Metrics

If you are adding a new backend or endpoint and want to track custom metrics, use the shared Prometheus registry:

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/mudler/LocalAI/pkg/metrics"
)

var myBackendRequests = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "localai_mybackend_requests_total",
        Help: "Total requests handled by mybackend",
    },
    []string{"model", "status"},
)

func init() {
    metrics.Register(myBackendRequests)
}
```

Then increment within your handler:

```go
myBackendRequests.WithLabelValues(modelName, "success").Inc()
```

---

## Distributed Tracing (OpenTelemetry)

### Enabling Tracing

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=localai
LOCALAI_OTEL_TRACING=true
```

LocalAI will automatically create spans for:
- Incoming HTTP requests (via middleware)
- Model load operations
- Backend inference calls

### Adding Spans to New Code

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
)

func MyInferenceFunction(ctx context.Context, modelName string) error {
    tracer := otel.Tracer("localai/mybackend")
    ctx, span := tracer.Start(ctx, "MyInferenceFunction")
    defer span.End()

    span.SetAttributes(attribute.String("model", modelName))

    // ... do work ...

    return nil
}
```

---

## Structured Logging

LocalAI uses `zerolog` for structured JSON logging. All log output is written to `stderr`.

### Log Levels

Set via environment variable:

```bash
LOCALAI_LOG_LEVEL=debug   # trace | debug | info | warn | error
```

### Logging Conventions

```go
import "github.com/rs/zerolog/log"

// Informational
log.Info().Str("model", modelName).Msg("Model loaded successfully")

// Warning
log.Warn().Str("backend", backendName).Int("retries", retryCount).Msg("Backend slow to respond")

// Error (always include the error object)
log.Error().Err(err).Str("model", modelName).Msg("Failed to run inference")

// Debug (use for verbose operational details)
log.Debug().Interface("request", req).Msg("Incoming inference request")
```

**Do not** use `fmt.Println` or `log.Printf` in new code — always use `zerolog`.

---

## Health Checks

| Endpoint | Description |
|---|---|
| `GET /readyz` | Returns 200 when the API is ready to serve traffic |
| `GET /healthz` | Returns 200 if the process is alive |

These are suitable for Kubernetes `readinessProbe` and `livenessProbe` configurations.

---

## Grafana Dashboard

A pre-built Grafana dashboard JSON is available at `extras/grafana/localai-dashboard.json`.

It includes panels for:
- Request rate and error rate
- p50/p95/p99 latency
- Active models
- Token throughput

Import it via **Grafana → Dashboards → Import → Upload JSON**.
