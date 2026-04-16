# Audio and Transcription in LocalAI

This document covers how to implement audio transcription (speech-to-text), text-to-speech (TTS), and related audio processing backends in LocalAI.

## Overview

LocalAI supports:
- **Speech-to-Text (STT)**: Transcribe audio files using Whisper or compatible backends
- **Text-to-Speech (TTS)**: Generate audio from text using piper, bark, or other backends
- **Voice Activity Detection (VAD)**: Detect speech segments in audio

## Directory Structure

```
core/
  backend/
    transcription.go   # STT logic
    tts.go             # TTS logic
api/
  openai/
    transcription.go   # /v1/audio/transcriptions handler
    tts.go             # /v1/audio/speech handler
```

## Transcription Handler

```go
// TranscriptionRequest mirrors the OpenAI audio transcription API.
type TranscriptionRequest struct {
    File           multipart.File
    Model          string
    Language       string
    Prompt         string
    ResponseFormat string // json, text, srt, verbose_json, vtt
    Temperature    float32
}

func TranscriptionEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Parse multipart form — file is required
        file, header, err := c.Request.FormFile("file")
        if err != nil {
            c.JSON(http.StatusBadRequest, ErrorResponse{Error: &APIError{Message: "file is required", Code: http.StatusBadRequest}})
            return
        }
        defer file.Close()

        modelName := c.PostForm("model")
        if modelName == "" {
            modelName = appConfig.DefaultModel
        }

        // 2. Save uploaded file to a temp path
        tmpFile, err := os.CreateTemp("", "localai-audio-*.wav")
        if err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: "failed to create temp file"}})
            return
        }
        defer os.Remove(tmpFile.Name())
        if _, err := io.Copy(tmpFile, file); err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: "failed to save audio"}})
            return
        }
        tmpFile.Close()

        log.Info().Str("model", modelName).Str("filename", header.Filename).Msg("transcription request")

        // 3. Load backend config
        cfg, err := cl.LoadBackendConfigFileByName(modelName, appConfig)
        if err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: err.Error()}})
            return
        }

        // 4. Run transcription
        result, err := backend.ModelTranscription(tmpFile.Name(), c.PostForm("language"), ml, cfg, appConfig)
        if err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: err.Error()}})
            return
        }

        // 5. Return result
        c.JSON(http.StatusOK, result)
    }
}
```

## Backend: ModelTranscription

```go
// ModelTranscription calls the underlying STT backend (e.g. whisper.cpp).
func ModelTranscription(audioPath, language string, ml *model.ModelLoader, cfg *config.BackendConfig, appConfig *config.ApplicationConfig) (*schema.TranscriptionResult, error) {
    opts := modelOptions(cfg, appConfig)
    whisperModel, err := ml.BackendLoader(backend.WhisperBackend, cfg.Model, opts...)
    if err != nil {
        return nil, fmt.Errorf("loading whisper model: %w", err)
    }

    tr, ok := whisperModel.(grpc.Backend)
    if !ok {
        return nil, fmt.Errorf("backend does not support transcription")
    }

    req := &proto.TranscriptRequest{
        Dst:      audioPath,
        Language: language,
        Threads:  uint32(cfg.Threads),
    }

    resp, err := tr.AudioTranscription(context.Background(), req)
    if err != nil {
        return nil, fmt.Errorf("transcription failed: %w", err)
    }

    return &schema.TranscriptionResult{
        Text:     resp.Text,
        Segments: toSegments(resp.Segments),
    }, nil
}
```

## TTS Handler

```go
// TTSEndpoint handles POST /v1/audio/speech
func TTSEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req schema.TTSRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, ErrorResponse{Error: &APIError{Message: err.Error()}})
            return
        }

        cfg, err := cl.LoadBackendConfigFileByName(req.Model, appConfig)
        if err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: err.Error()}})
            return
        }

        // Output to a temp file, then stream back
        tmpFile, err := os.CreateTemp("", "localai-tts-*.wav")
        if err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: "failed to create temp file"}})
            return
        }
        defer os.Remove(tmpFile.Name())
        tmpFile.Close()

        if err := backend.ModelTTS(req.Input, req.Voice, tmpFile.Name(), ml, cfg, appConfig); err != nil {
            c.JSON(http.StatusInternalServerError, ErrorResponse{Error: &APIError{Message: err.Error()}})
            return
        }

        c.Header("Content-Type", "audio/wav")
        c.File(tmpFile.Name())
    }
}
```

## Model Configuration for Audio

In your model YAML:

```yaml
name: whisper-1
backend: whisper
parameters:
  model: whisper-base.en
threads: 4
```

For TTS with piper:

```yaml
name: tts-1
backend: piper
parameters:
  model: en_US-lessac-medium
audio_path: /tmp/localai-tts
```

## Response Schemas

```go
type TranscriptionResult struct {
    Text     string            `json:"text"`
    Segments []TranscriptSegment `json:"segments,omitempty"`
}

type TranscriptSegment struct {
    ID    int     `json:"id"`
    Start float64 `json:"start"`
    End   float64 `json:"end"`
    Text  string  `json:"text"`
}

type TTSRequest struct {
    Model          string  `json:"model"`
    Input          string  `json:"input"`
    Voice          string  `json:"voice"`
    ResponseFormat string  `json:"response_format"` // wav, mp3, opus
    Speed          float32 `json:"speed"`
}
```

## Testing

```bash
# Transcription
curl http://localhost:8080/v1/audio/transcriptions \
  -F file=@audio.wav \
  -F model=whisper-1

# TTS
curl http://localhost:8080/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"model":"tts-1","input":"Hello world","voice":"alloy"}' \
  --output speech.wav
```

## Notes

- Always clean up temp audio files after processing
- Whisper backends expect 16kHz mono WAV; consider adding ffmpeg conversion if the input format varies
- TTS voice names are mapped in the backend config under `voice_map`
- For long audio, consider chunking at silence boundaries to stay within backend limits
