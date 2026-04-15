# Model Gallery and Management

This document describes how LocalAI manages model galleries, downloads, and lifecycle operations.

## Overview

LocalAI supports a model gallery system that allows users to browse, install, and manage AI models from curated repositories. The gallery system is built around a YAML-based model definition format and supports multiple sources.

## Gallery Configuration

Galleries are configured in the main LocalAI configuration file or via environment variables:

```yaml
galleries:
  - name: localai
    url: https://raw.githubusercontent.com/mudler/LocalAI/master/gallery/index.yaml
  - name: community
    url: https://raw.githubusercontent.com/your-org/models/main/index.yaml
```

Environment variable:
```bash
GALLERIES='[{"name":"localai","url":"https://..."}]'
```

## Gallery Index Format

Each gallery exposes an index YAML file listing available models:

```yaml
- name: llama-3.2-1b-instruct
  urls:
    - https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf
  sha256: "abc123..."
  config_url: https://raw.githubusercontent.com/.../llama-3.2-1b.yaml
  description: "Llama 3.2 1B Instruct model"
  license: llama3.2
  tags:
    - llm
    - instruct
    - llama
```

## Model Definition Files

Gallery model definitions (stored under `gallery/`) describe how to install a model:

```yaml
# gallery/llama-3.2-1b-instruct.yaml
name: llama-3.2-1b-instruct
description: Meta Llama 3.2 1B Instruct
license: llama3.2
urls:
  - https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf
files:
  - filename: llama-3.2-1b-instruct.gguf
    sha256: "deadbeef..."
    uri: https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf
config_file: |
  name: llama-3.2-1b-instruct
  backend: llama-cpp
  parameters:
    model: llama-3.2-1b-instruct.gguf
    context_size: 8192
    threads: 4template:
    chat: llama3-instruct
  stopwords:
    - "<|eot_id|>"
    - "<|end_of_text|>"
```

## API Endpoints

### List Available Models from Gallery

```
GET /models/available
```

Returns models from all configured galleries.

Response:
```json
[
  {
    "name": "llama-3.2-1b-instruct",
    "description": "Meta Llama 3.2 1B Instruct",
    "tags": ["llm", "instruct"],
    "installed": false,
    "license": "llama3.2"
  }
]
```

### Install a Model

```
POST /models/apply
Content-Type: application/json

{
  "id": "localai@llama-3.2-1b-instruct",
  "name": "my-llama"
}
```

The `id` field uses the format `gallery_name@model_name`. The optional `name` field overrides the installed model name.

Response (job started):
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "waiting"
}
```

### Check Installation Job Status

```
GET /models/jobs/{uuid}
```

Response:
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "downloading",
  "progress": 42.5,
  "file_name": "llama-3.2-1b-instruct.gguf",
  "downloaded_size": "1.2GB",
  "total_size": "2.8GB",
  "error": null
}
```

Status values: `waiting`, `downloading`, `processing`, `done`, `error`

### Delete a Model

```
POST /models/delete
Content-Type: application/json

{
  "name": "my-llama"
}
```

## Installation Process

1. **Resolve gallery**: Parse `gallery@model` identifier to find the gallery URL.
2. **Fetch definition**: Download and parse the model YAML definition.
3. **Download files**: Download model files with SHA256 verification.
4. **Write config**: Write the model configuration YAML to the models directory.
5. **Notify**: Signal the model loader to pick up the new model.

## Overriding Gallery Defaults

When calling `POST /models/apply`, you can override fields from the gallery definition:

```json
{
  "id": "localai@llama-3.2-1b-instruct",
  "name": "my-custom-llama",
  "overrides": {
    "parameters.threads": 8,
    "parameters.context_size": 4096
  }
}
```

Overrides use dot-notation to target nested config fields.

## Security Considerations

- Always verify SHA256 checksums after downloading model files.
- Gallery URLs should use HTTPS.
- When using custom galleries, review model definitions before installing.
- The `config_file` field in gallery definitions is written directly to disk — only use trusted gallery sources.

## Scanning Installed Models

LocalAI scans the models directory at startup and whenever a new model is installed. Files matching `*.yaml` or `*.yml` (excluding `*.gguf.yaml` sidecar files) are loaded as model configurations.

To force a rescan without restarting:
```
POST /models/reload
```

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `GALLERIES` | JSON array of gallery configs | `[]` |
| `MODELS_PATH` | Directory to store models and configs | `./models` |
| `GALLERY_FORCE_DOWNLOAD` | Re-download even if file exists | `false` |
| `PARALLEL_DOWNLOADS` | Max concurrent file downloads | `1` |
