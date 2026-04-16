# Prompt Templating in LocalAI

LocalAI uses Go's `text/template` package to render prompts before sending them to backends. This allows models to have custom chat formats (ChatML, Llama3, Mistral, etc.).

## Template Files

Templates are stored alongside model configs, typically in the models directory:

```
models/
  my-model.yaml
  my-model.tmpl          # chat template
  my-model-completion.tmpl  # completion template (optional)
```

## Template Variables

The following variables are available inside templates:

| Variable | Type | Description |
|---|---|---|
| `.Messages` | `[]Message` | Chat history |
| `.Input` | `string` | Raw user input (completion) |
| `.SystemPrompt` | `string` | System prompt if set |
| `.Functions` | `[]Function` | Tool/function definitions |

### Message Fields

```go
type Message struct {
    Role    string // "system", "user", "assistant", "tool"
    Content string
    Name    string // for tool results
}
```

## Example Templates

### ChatML

```
{{- range .Messages}}<|im_start|>{{.Role}}
{{.Content}}<|im_end|>
{{end}}<|im_start|>assistant
```

### Llama 3

```
{{- range .Messages}}<|start_header_id|>{{.Role}}<|end_header_id|>

{{.Content}}<|eot_id|>{{end}}<|start_header_id|>assistant<|end_header_id|>

```

### Mistral / Mixtral

```
{{- range .Messages}}{{if eq .Role "user"}}[INST] {{.Content}} [/INST]{{else if eq .Role "assistant"}} {{.Content}}</s>{{else if eq .Role "system"}}<<SYS>>
{{.Content}}
<</SYS>>

{{end}}{{end}}
```

## Model Config Reference

In the YAML config, point to templates:

```yaml
name: my-model
template:
  chat: my-model          # resolves to my-model.tmpl
  completion: my-model-completion
  # or inline:
  chat_message: |
    <|im_start|>{{.Role}}
    {{.Content}}<|im_end|>
```

## Rendering Templates in Code

```go
import "github.com/mudler/LocalAI/pkg/templates"

// Render a chat template
rendered, err := templates.RenderTemplate(templateName, templates.ChatTemplateData{
    Messages:     messages,
    SystemPrompt: systemPrompt,
    Functions:    functions,
})
if err != nil {
    return "", fmt.Errorf("template rendering failed: %w", err)
}
```

## Template Functions

Custom template functions available:

| Function | Usage | Description |
|---|---|---|
| `toJson` | `{{toJson .Functions}}` | Marshal value to JSON string |
| `add` | `{{add 1 2}}` | Integer addition |
| `isLastMessage` | `{{isLastMessage . $msgs}}` | True if last in slice |

## Debugging Templates

Enable template debug logging:

```bash
LOCAL_AI_DEBUG=true ./local-ai --log-level=debug
```

This will print the rendered prompt before each inference call.

## Common Issues

- **Extra newlines**: Use `{{-` and `-}}` to trim whitespace around blocks.
- **Missing EOS token**: Ensure your template ends with the correct stop token so the model knows when to stop generating.
- **Role mismatch**: Some models expect `human`/`bot` instead of `user`/`assistant` — remap in the template using `{{if eq .Role "user"}}human{{else}}bot{{end}}`.
