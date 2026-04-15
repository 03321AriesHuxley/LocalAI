# Error Handling and Logging in LocalAI

This guide covers best practices for error handling and structured logging across LocalAI backends and API layers.

## Logging Framework

LocalAI uses [zerolog](https://github.com/rs/zerolog) for structured, leveled logging. The logger is available via the `log` package alias throughout the codebase.

```go
import (
    "github.com/rs/zerolog/log"
)
```

## Log Levels

| Level   | Usage |
|---------|-------|
| `Trace` | Very fine-grained info, e.g., token-by-token streaming |
| `Debug` | Diagnostic info useful during development |
| `Info`  | General operational messages |
| `Warn`  | Unexpected but recoverable situations |
| `Error` | Errors that affect a single request but not the process |
| `Fatal` | Unrecoverable errors — calls `os.Exit(1)` |

## Structured Logging Examples

### Basic info log with context fields

```go
log.Info().
    Str("model", modelName).
    Str("backend", backendName).
    Msg("Model loaded successfully")
```

### Error with stack context

```go
if err != nil {
    log.Error().
        Err(err).
        Str("model", modelName).
        Str("request_id", reqID).
        Msg("inference failed")
    return nil, fmt.Errorf("inference error for model %s: %w", modelName, err)
}
```

### Debug log for tracing internal state

```go
log.Debug().
    Str("backend", b.Name).
    Int("tokens", tokenCount).
    Float64("temperature", opts.Temperature).
    Msg("starting generation")
```

## Error Wrapping Conventions

Always wrap errors with context using `fmt.Errorf` and the `%w` verb so callers can use `errors.Is` / `errors.As`:

```go
if err := backend.Load(cfg); err != nil {
    return fmt.Errorf("loading backend %q: %w", cfg.Backend, err)
}
```

Define sentinel errors for conditions callers need to distinguish:

```go
var (
    ErrModelNotFound   = errors.New("model not found")
    ErrBackendTimeout  = errors.New("backend request timed out")
    ErrContextExceeded = errors.New("context length exceeded")
)
```

Return them wrapped with additional detail:

```go
return fmt.Errorf("%w: requested %d, max %d", ErrContextExceeded, requested, maxCtx)
```

## HTTP Error Responses

Use the shared helper to return consistent OpenAI-compatible error JSON:

```go
// pkg/utils/http.go
func ErrorResponse(c *fiber.Ctx, status int, errType, message string) error {
    return c.Status(status).JSON(schema.ErrorResponse{
        Error: &schema.APIError{
            Type:    errType,
            Message: message,
        },
    })
}
```

Common patterns in route handlers:

```go
if err != nil {
    if errors.Is(err, ErrModelNotFound) {
        return ErrorResponse(c, fiber.StatusNotFound, "invalid_request_error", err.Error())
    }
    log.Error().Err(err).Str("path", c.Path()).Msg("unhandled handler error")
    return ErrorResponse(c, fiber.StatusInternalServerError, "server_error", "internal server error")
}
```

## Panic Recovery

Backend gRPC processes run in separate goroutines. Wrap long-running goroutines with a deferred recover to prevent a single backend crash from taking down the whole server:

```go
func safeGo(name string, fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Error().
                    Str("goroutine", name).
                    Interface("panic", r).
                    Msg("recovered from panic")
            }
        }()
        fn()
    }()
}
```

## Context Cancellation

Always propagate `context.Context` through inference calls and check for cancellation to avoid wasted compute:

```go
func (b *Backend) Infer(ctx context.Context, req *pb.PredictOptions) (*pb.Reply, error) {
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("inference cancelled: %w", ctx.Err())
    default:
    }
    // ... proceed with inference
}
```

## Logging in Tests

Set zerolog to a higher level in tests to reduce noise, or capture logs for assertion:

```go
func TestMain(m *testing.M) {
    zerolog.SetGlobalLevel(zerolog.WarnLevel)
    os.Exit(m.Run())
}
```

## Do's and Don'ts

**Do:**
- Always log the `model` and `backend` fields when available
- Use `%w` when wrapping errors
- Check `ctx.Done()` in long-running loops
- Return errors up the stack; let the handler decide the HTTP status

**Don't:**
- Use `log.Fatal` inside request handlers — it will kill the server
- Swallow errors silently with `_ = someCall()`
- Log sensitive data (API keys, user prompts at Info level)
- Use `fmt.Println` or `log` from the standard library
