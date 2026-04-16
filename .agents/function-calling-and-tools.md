# Function Calling and Tools in LocalAI

This guide covers how to implement and extend function calling (tools) support in LocalAI backends.

## Overview

LocalAI supports OpenAI-compatible function calling and tool use. Requests may include a `tools` or `functions` field, and the model may respond with `tool_calls` instead of plain content.

## Request Structures

```go
// ToolCall represents a single tool invocation from the model
type ToolCall struct {
    ID       string          `json:"id"`
    Type     string          `json:"type"` // always "function"
    Function FunctionCall    `json:"function"`
}

type FunctionCall struct {
    Name      string `json:"name"`
    Arguments string `json:"arguments"` // JSON-encoded string
}

// Tool defines an available tool in the request
type Tool struct {
    Type     string          `json:"type"`
    Function FunctionDefinition `json:"function"`
}

type FunctionDefinition struct {
    Name        string      `json:"name"`
    Description string      `json:"description"`
    Parameters  interface{} `json:"parameters"` // JSON Schema object
}
```

## Detecting Tool Use in a Request

```go
func hasTools(req *OpenAIRequest) bool {
    return len(req.Tools) > 0 || len(req.Functions) > 0
}

// Normalize: merge legacy Functions into Tools
func normalizeTools(req *OpenAIRequest) []Tool {
    tools := append([]Tool{}, req.Tools...)
    for _, f := range req.Functions {
        tools = append(tools, Tool{
            Type:     "function",
            Function: f,
        })
    }
    return tools
}
```

## Grammar-Based Constrained Decoding

Many backends (e.g. llama.cpp) support grammar constraints to force the model to output valid JSON matching a schema. LocalAI converts tool schemas to GBNF grammar strings.

```go
// BuildToolGrammar converts a list of tools into a GBNF grammar string
// that constrains model output to valid tool-call JSON.
func BuildToolGrammar(tools []Tool) (string, error) {
    if len(tools) == 0 {
        return "", nil
    }
    // Use json-schema-to-grammar conversion (see pkg/functions)
    schemas := make([]interface{}, len(tools))
    for i, t := range tools {
        schemas[i] = t.Function.Parameters
    }
    return jsonSchemaToGBNF(schemas)
}
```

Set the on the backend config before inference:

```go
if g, err := BuildToolGrammar(tools); err == nil && g != "" {
    config.Grammar = g
}
```

## Parsing Tool Calls from Model Output

After inference, parse raw model output into structured `ToolCall` objects:

```go
func ParseToolCalls(raw string) ([]ToolCall, error) {
    raw = strings.TrimSpace(raw)
    // Model may return a JSON object or array
    var calls []ToolCall
    if strings.HasPrefix(raw, "[") {
        if err := json.Unmarshal([]byte(raw), &calls); err != nil {
            return nil, err
        }
        return calls, nil
    }
    var single ToolCall
    if err := json.Unmarshal([]byte(raw), &single); err != nil {
        return nil, err
    }
    return []ToolCall{single}, nil
}
```

## Building the Response

When the model returns tool calls, set `finish_reason` to `"tool_calls"` and populate the `tool_calls` field instead of `content`:

```go
func buildToolCallChoice(calls []ToolCall) Choice {
    return Choice{
        Message: Message{
            Role:      "assistant",
            Content:   "",
            ToolCalls: calls,
        },
        FinishReason: "tool_calls",
        Index:        0,
    }
}
```

## Streaming Tool Calls

For streaming responses, tool call arguments are chunked across multiple delta events:

```go
func buildToolCallChunk(index int, id, name, argsDelta string) ChatCompletionStreamResponse {
    return ChatCompletionStreamResponse{
        Choices: []StreamChoice{{
            Index: 0,
            Delta: Message{
                ToolCalls: []ToolCall{{
                    Index:    index,
                    ID:       id,
                    Type:     "function",
                    Function: FunctionCall{Name: name, Arguments: argsDelta},
                }},
            },
        }},
    }
}
```

## Tool Choice

Respect the `tool_choice` field in the request:

- `"none"` — do not call any tool even if tools are provided
- `"auto"` (default) — model decides
- `{"type": "function", "function": {"name": "..."}}`  — force a specific tool

```go
func shouldEnableTools(req *OpenAIRequest) bool {
    if req.ToolChoice == "none" {
        return false
    }
    return has

## Testing

```go
func TestParseToolCalls(t *testing.T) {
    raw := `{"id":"call_1","type":"function","function":{"name":"get_weather","arguments":"{\"location\":\"Rome\"}"}}`
    calls, err := ParseToolCalls(raw)
    require.NoError(t, err)
    require.Len(t, calls, 1)
    assert.Equal(t, "get_weather", calls[0].Function.Name)
}
```

## Related Files

- `pkg/functions/` — grammar generation utilities
- `api/openai/chat.go` — chat completion handler integrating tool calls
- `.agents/streaming-and-sse.md` — streaming response patterns
