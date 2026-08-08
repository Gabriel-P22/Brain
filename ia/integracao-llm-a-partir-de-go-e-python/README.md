# Integração LLM a partir de Go e Python

## O LLM é só mais uma dependência de infra externa

Do ponto de vista de arquitetura, uma API de LLM não é conceitualmente diferente de qualquer outra API HTTP de terceiro (gateway de pagamento, serviço de e-mail): é I/O de rede, com latência, taxa de erro e custo por chamada. A armadilha mais comum é deixar chamada de LLM vazar para dentro da camada de domínio — o correto é tratá-la como **detalhe de infraestrutura**, atrás de uma interface pequena que o domínio define e a infra implementa, exatamente como já visto em [contexts/common/CLEAN-ARCHITECTURE.md](../../contexts/common/CLEAN-ARCHITECTURE.md) e no exemplo de repository pattern do [go/repository-pattern-na-pratica](../../go/repository-pattern-na-pratica/).

```go
// domain/llm.go — o domínio só conhece essa interface pequena
type Completer interface {
    Complete(ctx context.Context, prompt string) (string, error)
}
```

O domínio (regra de negócio) chama `Completer`, nunca `anthropic.Client` diretamente — trocar de provedor, mockar em teste, ou adicionar cache vira detalhe de infra, sem tocar em regra de negócio. Mesmo raciocínio de DIP (Dependency Inversion) já visto em [go/solid-em-go](../../go/solid-em-go/).

## Go: `net/http` + `context.Context`

```go
package llmclient

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type AnthropicClient struct {
    apiKey     string
    httpClient *http.Client
}

func NewAnthropicClient(apiKey string) *AnthropicClient {
    return &AnthropicClient{
        apiKey:     apiKey,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}

func (c *AnthropicClient) Complete(ctx context.Context, prompt string) (string, error) {
    body, err := json.Marshal(map[string]any{
        "model":      "claude-sonnet-5",
        "max_tokens": 1024,
        "messages":   []map[string]string{{"role": "user", "content": prompt}},
    })
    if err != nil {
        return "", fmt.Errorf("montar request: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPost,
        "https://api.anthropic.com/v1/messages", bytes.NewReader(body))
    if err != nil {
        return "", fmt.Errorf("criar request: %w", err)
    }
    req.Header.Set("x-api-key", c.apiKey)
    req.Header.Set("content-type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return "", fmt.Errorf("chamar API de LLM: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("API de LLM retornou status %d", resp.StatusCode)
    }
    // ... decodificar resp.Body no shape de resposta esperado
    return "", nil
}
```

Pontos que já são familiares de [go/acesso-a-dados-e-context](../../go/acesso-a-dados-e-context/) e se aplicam aqui sem alteração:
- **`context.Context` propaga timeout/cancelamento** — se o handler HTTP que originou a chamada for cancelado (cliente desconectou), a chamada ao provedor de LLM é cancelada junto, em vez de continuar consumindo tempo/dinheiro para uma resposta que ninguém vai receber.
- **`fmt.Errorf` com `%w`** preserva a cadeia de erro — quem chama `Complete` consegue distinguir "erro de rede" de "erro 4xx do provedor" via `errors.Is`/`errors.As`, sem re-parsear string de erro.
- Um SDK oficial (quando existe para Go) resolve isso pronto — o `net/http` manual aqui é para deixar explícito o que o SDK faz por baixo.

## Python

```python
import anthropic
from anthropic import APIError, APIConnectionError, RateLimitError

client = anthropic.Anthropic(api_key=api_key, timeout=30.0)

def complete(prompt: str) -> str:
    try:
        response = client.messages.create(
            model="claude-sonnet-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text
    except RateLimitError:
        raise  # deixa o chamador decidir retry (ver abaixo)
    except (APIConnectionError, APIError) as e:
        raise RuntimeError(f"falha ao chamar LLM: {e}") from e
```

Diferença de postura em relação a Go: exceções concentram o tratamento de erro no `try/except`, contra o `if err != nil` explícito a cada chamada em Go — mesmo trade-off já coberto em [go/tratamento-de-erros-e-pacotes](../../go/tratamento-de-erros-e-pacotes/), só que agora aplicado a uma dependência externa em vez de lógica interna.

## Erro, retry e rate limit

Chamada de LLM falha por motivos que não existiam (ou eram raros) em chamada a um banco local: rate limit do provedor (`429`), timeout de rede, erro transiente `5xx`. Retry automático só faz sentido para esses casos — nunca para erro `4xx` de validação (prompt/schema errado), que vai falhar de novo do mesmo jeito.

```python
import time

def complete_with_retry(prompt: str, max_attempts: int = 3) -> str:
    for attempt in range(1, max_attempts + 1):
        try:
            return complete(prompt)
        except RateLimitError:
            if attempt == max_attempts:
                raise
            time.sleep(2 ** attempt)  # backoff exponencial
```

Em Go, o mesmo padrão de backoff exponencial se aplica em cima do `Completer`, geralmente como um decorator/wrapper que implementa a mesma interface (`RetryingCompleter` envolvendo um `AnthropicClient`) — de novo, composição em vez de herança, como já visto em [go/solid-em-go](../../go/solid-em-go/).

## Onde isso se encaixa na arquitetura

```
domain/          → define Completer (interface), regra de negócio usa só isso
infra/llm/       → AnthropicClient implementa Completer, trata retry/timeout/erro de provedor
infra/llm/cache/ → opcional: decorator de cache em cima de Completer (prompt caching do lado do app)
```

Isso fecha o ciclo com [APIs de LLM e streaming](../apis-de-llm-e-streaming/) (o que a chamada faz) e com o Módulo 3 de Go ([go/net-http-e-apis-rest](../../go/net-http-e-apis-rest/), [go/middleware-e-chain-de-handlers](../../go/middleware-e-chain-de-handlers/)) — um serviço que expõe endpoint HTTP e por baixo chama LLM é a mesma arquitetura em camadas já estudada, só trocando o repositório de banco de dados por um repositório de "geração de texto".
