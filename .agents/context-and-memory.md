# Context and Memory Management in LocalAI

This guide covers how LocalAI manages conversation context, token limits, and in-memory state for inference backends.

## Overview

LocalAI supports stateful conversation history through a context window abstraction. Each request can carry a `messages` array (OpenAI-compatible) or a raw `prompt`. The core logic for managing context lives in `pkg/model` and `core/context`.

## Key Concepts

### Context Window

Every model has a `context_size` (alias: `ctx_size`) parameter in its YAML config:

```yaml
name: my-model
backend: llama-cpp
parameters:
  model: models/my-model.gguf
context_size: 4096
```

If not set, the backend default is used (often 512 or 2048 depending on the backend).

### Message History Truncation

When the accumulated token count of `messages` exceeds `context_size`, LocalAI truncates oldest messages (excluding the system prompt) to fit within the window.

The truncation helper lives in `pkg/utils/tokens.go`:

```go
// TruncateMessages removes oldest non-system messages until the
// estimated token count fits within maxTokens.
func TruncateMessages(msgs []schema.Message, maxTokens int) []schema.Message {
    for EstimateTokens(msgs) > maxTokens && len(msgs) > 1 {
        // Preserve index-0 system message if present
        if msgs[0].Role == "system" && len(msgs) > 2 {
            msgs = append(msgs[:1], msgs[2:]...)
        } else {
            msgs = msgs[1:]
        }
    }
    return msgs
}
```

### Token Estimation

Because tokenizers differ per model, LocalAI uses a fast heuristic (4 chars ≈ 1 token) for truncation decisions unless a model exposes a `/tokenize` endpoint:

```go
func EstimateTokens(msgs []schema.Message) int {
    total := 0
    for _, m := range msgs {
        total += len(m.Content) / 4
    }
    return total
}
```

For exact counts, use the `TokenizeRequest` path which calls the backend's native tokenizer.

## In-Memory Session Store

LocalAI does **not** persist conversation history between process restarts by default. Sessions are held in a `sync.Map` keyed by a caller-supplied `session_id`:

```go
var sessions sync.Map // map[string][]schema.Message

func GetSession(id string) []schema.Message {
    if v, ok := sessions.Load(id); ok {
        return v.([]schema.Message)
    }
    return nil
}

func SetSession(id string, msgs []schema.Message) {
    sessions.Store(id, msgs)
}

func DeleteSession(id string) {
    sessions.Delete(id)
}
```

To enable session tracking, pass `"session_id": "<uuid>"` in the request body (LocalAI extension field).

## Configuration Options

| Field | Type | Default | Description |
|---|---|---|---|
| `context_size` | int | backend default | Max tokens in context window |
| `max_tokens` | int | 512 | Max tokens to generate |
| `session_id` | string | "" | Opaque key for history tracking |
| `keep_history` | bool | false | Persist messages across calls |

## Adding Context Support to a New Backend

1. Read `context_size` from `ModelConfig`:
   ```go
   ctxSize := config.ContextSize
   if ctxSize == 0 {
       ctxSize = 2048 // your backend default
   }
   ```

2. Before calling the model, truncate messages:
   ```go
   req.Messages = utils.TruncateMessages(req.Messages, ctxSize-req.MaxTokens)
   ```

3. Flatten messages to a single prompt string using `pkg/templates`:
   ```go
   prompt, err := templates.RenderChatPrompt(config, req.Messages)
   ```

## Prompt Templates

Templates live under `models/` or can be embedded in the YAML config:

```yaml
template:
  chat: |
    {{- range .Messages }}
    {{ .Role }}: {{ .Content }}
    {{- end }}
    assistant:
```

The `RenderChatPrompt` function executes the Go `text/template` with the messages slice.

## KV Cache (llama.cpp)

For `llama-cpp` backends, the KV cache is managed inside the C++ layer. LocalAI exposes:

- `cache_prompt: true` — reuse KV cache across requests with identical prefix
- `cache_type_k` / `cache_type_v` — quantization type for K/V cache (`f16`, `q8_0`, `q4_0`)

```yaml
llama_cpp_params:
  cache_prompt: true
  cache_type_k: q8_0
  cache_type_v: q8_0
```

This significantly reduces TTFT (time-to-first-token) for multi-turn conversations.

## Debugging Context Issues

- Enable `debug: true` in the model config to log prompt length before inference.
- Watch for `context length exceeded` errors — increase `context_size` or enable truncation.
- Use the `/tokenize` endpoint to get exact token counts for a given input.

## Related Files

- `pkg/utils/tokens.go` — token estimation and truncation
- `pkg/templates/` — chat prompt rendering
- `core/schema/` — `Message` and `ChatCompletionRequest` types
- `core/context/` — session store and request context helpers
- `.agents/model-configuration.md` — full model YAML reference
- `.agents/debugging-backends.md` — debugging inference issues
