# Embedding and Reranking in LocalAI

This guide covers how embedding and reranking backends work in LocalAI, how to implement new ones, and how requests flow through the system.

## Overview

LocalAI supports two related inference types:
- **Embeddings**: Convert text into dense vector representations
- **Reranking**: Score and reorder documents by relevance to a query

Both share similar backend plumbing but have distinct API shapes.

## Embedding Request Flow

```
POST /v1/embeddings
  → EmbeddingsEndpoint (api/endpoints)
  → modelLoader.GetModel()
  → backend.Embeddings(req)
  → []float32 vector returned
```

## Reranking Request Flow

```
POST /v1/rerank
  → RerankEndpoint
  → modelLoader.GetModel()
  → backend.Rerank(query, documents)
  → scored + sorted document list
```

## Implementing an Embedding Backend

A backend that supports embeddings must implement the `EmbeddingsBackend` interface:

```go
type EmbeddingsBackend interface {
    Embeddings(text string) ([]float32, error)
}
```

Example minimal implementation:

```go
func (b *MyBackend) Embeddings(text string) ([]float32, error) {
    req := &pb.EmbeddingRequest{Text: text}
    resp, err := b.client.Embeddings(context.Background(), req)
    if err != nil {
        return nil, fmt.Errorf("embeddings inference failed: %w", err)
    }
    return resp.Embeddings, nil
}
```

## Implementing a Reranking Backend

```go
type RerankBackend interface {
    Rerank(query string, documents []string) ([]RankedDocument, error)
}

type RankedDocument struct {
    Index          int     `json:"index"`
    RelevanceScore float64 `json:"relevance_score"`
    Document       *string `json:"document,omitempty"`
}
```

Example:

```go
func (b *MyBackend) Rerank(query string, docs []string) ([]RankedDocument, error) {
    req := &pb.RerankRequest{Query: query, Documents: docs}
    resp, err := b.client.Rerank(context.Background(), req)
    if err != nil {
        return nil, fmt.Errorf("rerank failed: %w", err)
    }
    ranked := make([]RankedDocument, len(resp.Results))
    for i, r := range resp.Results {
        ranked[i] = RankedDocument{
            Index:          int(r.Index),
            RelevanceScore: float64(r.Score),
        }
    }
    return ranked, nil
}
```

## Model Configuration for Embeddings

In your model YAML, set:

```yaml
name: my-embedding-model
backend: bert-embeddings   # or llama-cpp, etc.
embeddings: true
parameters:
  model: /models/my-model.bin
```

For reranking models:

```yaml
name: my-reranker
backend: rerankers
parameters:
  model: /models/reranker.bin
```

## Normalization

Many embedding use cases require L2-normalized vectors. LocalAI does **not** normalize by default. If your use case requires it, normalize in the client or add a post-processing step:

```go
func normalizeVector(v []float32) []float32 {
    var sum float64
    for _, x := range v {
        sum += float64(x) * float64(x)
    }
    norm := float32(math.Sqrt(sum))
    if norm == 0 {
        return v
    }
    out := make([]float32, len(v))
    for i, x := range v {
        out[i] = x / norm
    }
    return out
}
```

## Batching

The `/v1/embeddings` endpoint accepts both a single string and an array:

```json
{ "input": "single string", "model": "my-model" }
{ "input": ["string one", "string two"], "model": "my-model" }
```

Internally, each input is processed sequentially unless the backend explicitly supports batch inference. Check `backend.SupportsBatch()` before attempting parallel dispatch.

## Error Handling

- Return HTTP 400 if `input` is empty or missing
- Return HTTP 422 if the model does not support embeddings (`embeddings: true` not set)
- Return HTTP 500 for backend inference errors

Always wrap backend errors with context:

```go
if err != nil {
    return fmt.Errorf("embeddings(%s): %w", modelName, err)
}
```

## Testing

```bash
curl http://localhost:8080/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"input": "hello world", "model": "my-embedding-model"}'
```

Expected response shape:

```json
{
  "object": "list",
  "data": [
    { "object": "embedding", "index": 0, "embedding": [0.1, -0.3, ...] }
  ],
  "model": "my-embedding-model",
  "usage": { "prompt_tokens": 3, "total_tokens": 3 }
}
```
