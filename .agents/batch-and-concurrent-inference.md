# Batch and Concurrent Inference in LocalAI

This guide covers patterns for handling multiple inference requests concurrently, batching strategies, and managing backend worker pools.

## Overview

LocalAI uses a slot-based concurrency model. Each model backend has a configurable number of parallel slots. Requests queue when all slots are busy.

## Key Concepts

### Model Mutex / Slot Management

Each loaded model is guarded by a semaphore derived from `config.Threads` or `config.ParallelRequests`:

```go
// Acquire a slot before inference
if err := model.Semaphore.Acquire(ctx, 1); err != nil {
    return nil, fmt.Errorf("acquiring inference slot: %w", err)
}
defer model.Semaphore.Release(1)
```

Set `parallel_requests: 4` in the model YAML to allow up to 4 concurrent requests against the same backend process.

### Request Queuing

When a request cannot acquire a slot within the deadline it returns HTTP 503. Clients should implement exponential back-off:

```
Retry-After: 2
X-Queue-Position: 3
```

## Batch Endpoint Pattern

For workloads that submit many prompts at once, process them with bounded parallelism:

```go
func RunBatch(ctx context.Context, requests []InferRequest, maxWorkers int) ([]InferResult, error) {
    results := make([]InferResult, len(requests))
    sem := make(chan struct{}, maxWorkers)
    var wg sync.WaitGroup
    var mu sync.Mutex
    var firstErr error

    for i, req := range requests {
        wg.Add(1)
        go func(idx int, r InferRequest) {
            defer wg.Done()
            sem <- struct{}{}
            defer func() { <-sem }()

            res, err := runSingleInference(ctx, r)
            mu.Lock()
            defer mu.Unlock()
            if err != nil && firstErr == nil {
                firstErr = err
            }
            results[idx] = res
        }(i, req)
    }

    wg.Wait()
    return results, firstErr
}
```

## Streaming vs Batch Trade-offs

| Mode | Latency | Throughput | Use case |
|------|---------|------------|----------|
| Streaming SSE | Low TTFT | Lower | Chat UI |
| Batch (non-stream) | Higher | Higher | Offline jobs |
| Parallel slots | Medium | Medium | API serving |

## Configuration

```yaml
# model.yaml
name: my-model
parallel_requests: 4   # concurrent slots
timeout: 120           # seconds before slot acquisition fails
threads: 8             # passed to backend
```

## Context Cancellation

Always propagate the HTTP request context so that cancelled clients free their slot immediately:

```go
func inferWithContext(ctx context.Context, model *Model, prompt string) (string, error) {
    ch := make(chan inferResult, 1)
    go func() {
        out, err := model.backend.Predict(prompt)
        ch <- inferResult{out, err}
    }()

    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case res := <-ch:
        return res.text, res.err
    }
}
```

## Metrics

Track queue depth and slot utilisation via the Prometheus metrics described in `observability-and-metrics.md`:

- `localai_active_slots{model}` — gauge of slots in use
- `localai_queued_requests{model}` — gauge of waiting requests
- `localai_inference_duration_seconds{model}` — histogram

## Common Pitfalls

- **Deadlock**: never call another model from inside a slot-held goroutine without releasing first.
- **Slot leak**: always use `defer sem.Release(1)`; recover panics with `safeGo` (see `error-handling-and-logging.md`).
- **Context ignored**: backends that block in C/CGO must check `ctx.Done()` in a separate goroutine and abort via the backend's cancel API.
