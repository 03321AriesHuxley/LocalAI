# Model Configuration in LocalAI

This guide covers how models are configured, loaded, and managed in LocalAI.

## Configuration File Format

Models are configured via YAML files placed in the models directory (default: `./models`).

### Basic Structure

```yaml
name: my-model
backend: llama-cpp
parameters:
  model: model-filename.gguf
  context_size: 4096
  threads: 4
  f16: true

# Template for prompt formatting
template:
  chat: |
    {{.Input}}
  completion: |
    {{.Input}}
```

## Key Configuration Fields

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Model alias used in API requests |
| `backend` | string | Backend to use (e.g., `llama-cpp`, `whisper`, `diffusers`) |
| `parameters` | object | Backend-specific parameters |
| `template` | object | Prompt templates for different endpoints |
| `system_prompt` | string | Default system prompt |
| `roles` | object | Role name mappings |

### Parameters Block

```yaml
parameters:
  model: path/to/model.gguf   # relative to models dir
  context_size: 2048
  seed: -1
  threads: 0                  # 0 = auto-detect
  f16: true
  mmap: true
  mlock: false
  gpu_layers: 0               # layers to offload to GPU
  low_vram: false
  numa: false
  embeddings: false           # enable embeddings endpoint
```

### Template Rendering

Templates use Go's `text/template` syntax. Available variables:

- `{{.Input}}` — user input text
- `{{.Instruction}}` — instruction (for instruction-tuned models)
- `{{.System}}` — system prompt
- `{{.Response}}` — model response (used in few-shot templates)

#### Chat Template Example (ChatML)

```yaml
template:
  chat_message: |
    <|im_start|>{{.RoleName}}
    {{.Content}}<|im_end|>
  chat: |
    {{.Input}}<|im_start|>assistant
```

## Feature Flags

```yaml
# Enable specific API features
function_call_results: true
stop_words:
  - "</s>"
  - "<|im_end|>"

# Usage tracking
usage: true

# Disable specific endpoints for this model
disabled_endpoints:
  - /v1/images/generations
```

## Grammar / JSON Mode

To constrain outputs to valid JSON or a specific grammar:

```yaml
parameters:
  grammar_asset: json.gbnf   # file in models dir
```

Or inline:

```yaml
parameters:
  grammar: |
    root ::= object
    object ::= "{" ...
```

## Multimodal Models

For vision-capable models:

```yaml
backend: llama-cpp
parameters:
  model: llava-v1.5-7b-q4.gguf
  mmproj: llava-v1.5-7b-mmproj-f16.gguf
```

## Loading Priority

LocalAI resolves model configs in this order:

1. Exact filename match in models directory
2. YAML config with matching `name` field
3. Gallery-installed model configs
4. Auto-detected model file (no config required for basic use)

## Environment Variable Overrides

Some parameters can be overridden via environment variables:

```bash
LOCALAI_THREADS=8           # override default thread count
LOCALAI_CONTEXT_SIZE=8192   # override default context size
LOCALAI_F16=true            # force f16 mode
```

## Validating a Config

Use the LocalAI CLI to validate before deploying:

```bash
local-ai validate-config ./models/my-model.yaml
```

## See Also

- [Adding Backends](.agents/adding-backends.md)
- [Adding Gallery Models](.agents/adding-gallery-models.md)
- [Debugging Backends](.agents/debugging-backends.md)
