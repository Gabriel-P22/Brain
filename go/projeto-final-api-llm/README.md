# Projeto final — API + LLM

Síntese do Módulo 3 inteiro: um serviço Go pequeno que expõe uma rota HTTP e, por trás, chama uma API de LLM externa — junta API REST, JSON, `context` com timeout, error handling, config via env var e (opcional) concorrência.

## Esboço de arquitetura

```
llmproxy/
  main.go               // monta tudo, sobe o servidor
  domain/
    chat.go               // ChatRequest, ChatResponse — tipos de domínio, sem saber de HTTP nem do provedor de LLM
  llm/
    client.go              // interface LLMClient — abstração do provedor
  infra/
    anthropic/
      client.go              // implementa LLMClient chamando a API real
  http/
    chat_handler.go           // recebe POST /chat, valida, chama o use case, responde
```

```go
// llm/client.go — domínio não conhece o provedor concreto
type LLMClient interface {
    Complete(ctx context.Context, prompt string) (string, error)
}

// infra/anthropic/client.go — só aqui entra a chamada HTTP real pro provedor
type Client struct {
    apiKey string
    http   *http.Client
}

func (c *Client) Complete(ctx context.Context, prompt string) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, http.MethodPost, apiURL, buildBody(prompt))
    req.Header.Set("Authorization", "Bearer "+c.apiKey)

    resp, err := c.http.Do(req)
    if err != nil {
        return "", fmt.Errorf("chamando LLM: %w", err)
    }
    defer resp.Body.Close()

    var result Response
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return "", fmt.Errorf("decodificando resposta do LLM: %w", err)
    }
    return result.Text, nil
}
```

```go
// http/chat_handler.go — camada de entrega, HTTP + LLM não se misturam com o domínio
func chatHandler(client llm.LLMClient) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req domain.ChatRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "corpo inválido", http.StatusBadRequest)
            return
        }
        text, err := client.Complete(r.Context(), req.Prompt)
        if err != nil {
            writeError(w, err)   // ver Tratamento de erros e exceções em APIs
            return
        }
        json.NewEncoder(w).Encode(domain.ChatResponse{Text: text})
    }
}
```

## O que isso amarra, do plano inteiro

- `context.WithTimeout` — [Acesso a dados e context.Context](../acesso-a-dados-e-context/): nunca deixar a chamada externa travar pra sempre.
- Interface `LLMClient` declarada perto de quem usa, implementação concreta separada — [SOLID em Go](../solid-em-go/) (DIP) + [Clean Architecture](../clean-architecture-separacao-em-camadas/).
- Erro de rede/provedor virando resposta HTTP consistente — [Tratamento de erros e exceções em APIs](../tratamento-de-erros-e-excecoes-em-apis/).
- Chave de API via variável de ambiente, nunca hardcoded — [Produção — logging, config, error wrapping](../producao-logging-config-error-wrapping/) e [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#configuração).
- Testável com um `FakeLLMClient` no lugar do `anthropic.Client` real — [Testes](../testes-unitarios-e-integracao/).

Extensão natural se quiser ir além: worker pool pra processar N prompts em paralelo respeitando rate limit do provedor ([Padrões de concorrência para infra](../padroes-de-concorrencia-para-infra/)).

Sem exercício pré-gerado — implementação real fica pra quando você rodar `/exercise projeto final` ou pedir direto no chat.
