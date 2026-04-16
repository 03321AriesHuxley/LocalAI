# Multimodal and Vision Support in LocalAI

This guide covers how to handle multimodal inputs (images, audio) in LocalAI backends and API handlers.

## Overview

LocalAI supports multimodal models that can process both text and images (e.g., LLaVA, BakLLaVA, GPT-4V-compatible APIs). Vision inputs arrive via the OpenAI-compatible chat completions endpoint as `image_url` content parts.

## Request Structure

Multimodal requests use the standard chat completions format with array-type `content`:

```json
{
  "model": "llava",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "What is in this image?"},
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}
      ]
    }
  ]
}
```

## Parsing Multimodal Content

In `pkg/model/multimodal.go` (or similar), content parts are extracted before inference:

```go
// ExtractVisionContent separates text and image parts from a multimodal message.
func ExtractVisionContent(content interface{}) (text string, images []string, err error) {
    switch v := content.(type) {
    case string:
        return v, nil, nil
    case []interface{}:
        for _, part := range v {
            partMap, ok := part.(map[string]interface{})
            if !ok {
                continue
            }
            switch partMap["type"] {
            case "text":
                text += fmt.Sprintf("%v", partMap["text"])
            case "image_url":
                if imgURL, ok := partMap["image_url"].(map[string]interface{}); ok {
                    images = append(images, fmt.Sprintf("%v", imgURL["url"]))
                }
            }
        }
        return text, images, nil
    }
    return "", nil, fmt.Errorf("unsupported content type: %T", content)
}
```

## Image Handling

Images arrive as either:
- **Base64 data URLs**: `data:image/png;base64,<data>`
- **HTTP URLs**: fetched at inference time

```go
// ResolveImage downloads or decodes an image to a local temp file path.
// Returns the file path and a cleanup function.
func ResolveImage(ctx context.Context, imageURL string) (string, func(), error) {
    if strings.HasPrefix(imageURL, "data:") {
        // Parse base64 data URL
        parts := strings.SplitN(imageURL, ",", 2)
        if len(parts) != 2 {
            return "", nil, fmt.Errorf("invalid data URL")
        }
        data, err := base64.StdEncoding.DecodeString(parts[1])
        if err != nil {
            return "", nil, fmt.Errorf("base64 decode: %w", err)
        }
        f, err := os.CreateTemp("", "localai-img-*.png")
        if err != nil {
            return "", nil, err
        }
        if _, err := f.Write(data); err != nil {
            f.Close()
            os.Remove(f.Name())
            return "", nil, err
        }
        f.Close()
        return f.Name(), func() { os.Remove(f.Name()) }, nil
    }
    // HTTP URL
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, imageURL, nil)
    if err != nil {
        return "", nil, err
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", nil, err
    }
    defer resp.Body.Close()
    f, err := os.CreateTemp("", "localai-img-*.png")
    if err != nil {
        return "", nil, err
    }
    if _, err := io.Copy(f, resp.Body); err != nil {
        f.Close()
        os.Remove(f.Name())
        return "", nil, err
    }
    f.Close()
    return f.Name(), func() { os.Remove(f.Name()) }, nil
}
```

## Model Configuration for Vision

In the model YAML, enable multimodal support:

```yaml
name: llava-1.5
backend: llama-cpp
parameters:
  model: llava-1.5-7b-Q4_K_M.gguf
multimodal: true
mmproj: mmproj-llava-1.5-7b-f16.gguf  # vision projection weights
```

The `mmproj` field specifies the multimodal projector model file, placed in the same models directory.

## Backend Integration

Backends that support vision (e.g., llama-cpp) receive image file paths via the `ImageData` field in the prediction request:

```go
// In the prediction options passed to the backend:
opts.ImageData = []schema.ImageData{
    {URL: resolvedImagePath},
}
```

The llama-cpp backend passes these to `llama_eval` with the clip model loaded via the `mmproj` path.

## Checking Vision Capability

Before processing, verify the loaded model supports vision:

```go
func modelSupportsVision(cfg *config.BackendConfig) bool {
    return cfg.Multimodal && cfg.MMProj != ""
}
```

If images are sent to a non-vision model, log a warning and strip image parts rather than returning an error, to maintain compatibility.

## Audio Inputs

Audio multimodal support (e.g., whisper-based models) follows the same pattern but uses `input_audio` content parts. Audio files are resolved similarly to images and passed to the backend as WAV/MP3 temp files.

## Testing

```bash
# Test vision with a local image
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "llava",
    "messages": [{"role": "user", "content": [
      {"type": "text", "text": "Describe this image"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBORw0KGgo="}}
    ]}]
  }'
```
