# Streaming and Server-Sent Events (SSE) in LocalAI

This guide covers how to implement and work with streaming responses and Server-Sent Events (SSE) in LocalAI backends and API endpoints.

## Overview

LocalAI supports streaming responses for inference endpoints (chat completions, completions, etc.) following the OpenAI streaming API format. Streaming uses SSE (Server-Sent Events) to push tokens to the client as they are generated.

## SSE Format

Each streamed chunk is sent as:

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"delta":{"content":"token"}}]}

data: [DONE]
```

The stream ends with `data: [DONE]`.

## Fiber SSE Handler Pattern

```go
package api

import (
    "bufio"
    "encoding/json"
    "fmt"

    "github.com/gofiber/fiber/v2"
    "github.com/mudler/LocalAI/core/schema"
    "github.com/rs/zerolog/log"
)

// streamResponse writes SSE chunks to a Fiber response.
// The channel receives partial schema.OpenAIResponse objects.
func streamResponse(c *fiber.Ctx, ch <-chan schema.OpenAIResponse) error {
    c.Set("Content-Type", "text/event-stream")
    c.Set("Cache-Control", "no-cache")
    c.Set("Connection", "keep-alive")
    c.Set("Transfer-Encoding", "chunked")

    c.Context().SetBodyStreamWriter(func(w *bufio.Writer) {
        for resp := range ch {
            data, err := json.Marshal(resp)
            if err != nil {
                log.Error().Err(err).Msg("failed to marshal streaming response")
                continue
            }
            fmt.Fprintf(w, "data: %s\n\n", data)
            if err := w.Flush(); err != nil {
                log.Warn().Err(err).Msg("client disconnected during stream")
                return
            }
        }
        // Signal end of stream
        fmt.Fprint(w, "data: [DONE]\n\n")
        w.Flush()
    })

    return nil
}
```

## Backend Streaming Interface

Backends that support streaming implement the `TokenCallback` pattern:

```go
// TokenCallback is called for each generated token during streaming inference.
// Return false to abort generation early (e.g., client disconnected).
type TokenCallback func(token string, usage backend.TokenUsage) bool

// Example backend call with streaming
func runStreamingInference(
    model backend.Model,
    req *schema.OpenAIRequest,
    cb TokenCallback,
) error {
    return model.Predict(req.Prompt, backend.PredictOptions{
        Tokens:        req.MaxTokens,
        TokenCallback: cb,
    })
}
```

## Wiring Streaming in a Handler

```go
func ChatCompletionHandler(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        req := new(schema.OpenAIRequest)
        if err := c.BodyParser(req); err != nil {
            return fiber.ErrBadRequest
        }

        if req.Stream {
            ch := make(chan schema.OpenAIResponse)
            go func() {
                defer close(ch)
                // Run inference, pushing chunks into ch
                err := runInferenceStreaming(req, cl, ml, appConfig, ch)
                if err != nil {
                    log.Error().Err(err).Msg("streaming inference error")
                }
            }()
            return streamResponse(c, ch)
        }

        // Non-streaming path
        resp, err := runInference(req, cl, ml, appConfig)
        if err != nil {
            return err
        }
        return c.JSON(resp)
    }
}
```

## Client Disconnect Detection

Always check for client disconnection inside the token callback:

```go
cb := func(token string, usage backend.TokenUsage) bool {
    select {
    case <-c.Context().Done():
        log.Debug().Msg("client disconnected, aborting stream")
        return false // abort generation
    default:
    }
    ch <- buildChunk(token, requestID)
    return true // continue
}
```

## Building a Chunk Response

```go
func buildChunk(token, id, model string) schema.OpenAIResponse {
    return schema.OpenAIResponse{
        ID:      id,
        Object:  "chat.completion.chunk",
        Model:   model,
        Choices: []schema.Choice{
            {
                Delta: &schema.Message{
                    Role:    "assistant",
                    Content: token,
                },
                Index:        0,
                FinishReason: "",
            },
        },
    }
}

// The final chunk should include finish_reason
func buildFinalChunk(id, model string, usage *schema.OpenAIUsage) schema.OpenAIResponse {
    resp := buildChunk("", id, model)
    resp.Choices[0].FinishReason = "stop"
    resp.Usage = usage
    return resp
}
```

## Testing Streaming Endpoints

Use `curl` to test:

```bash
curl -N http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gpt-4","stream":true,"messages":[{"role":"user","content":"Hello"}]}'
```

Or with the OpenAI Python SDK:

```python
for chunk in client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}],
    stream=True,
):
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

## Common Pitfalls

- **Forgetting to close the channel**: Always `defer close(ch)` in the goroutine sending chunks.
- **Not flushing the writer**: Call `w.Flush()` after every `data:` line.
- **Race on context cancellation**: Check `c.Context().Done()` inside the token callback.
- **Missing `[DONE]` sentinel**: Some clients require this to terminate the stream properly.
- **Large token batches**: Send one token at a time for best UX; batching delays perceived responsiveness.
