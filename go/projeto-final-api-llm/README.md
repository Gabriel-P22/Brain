# Projeto final — API + LLM

Síntese do Módulo 3 inteiro: um serviço Go pequeno que expõe uma rota HTTP e, por trás, chama uma API de LLM (modelo de linguagem) externa — junta API REST, JSON, `context` com timeout, tratamento de erro, configuração via variável de ambiente e, opcionalmente, concorrência.

## O problema de organização: por que não colocar tudo num arquivo só

O jeito mais rápido de escrever isso seria um único `main.go` com tudo dentro: ler a requisição, montar a chamada HTTP pro provedor de LLM, decodificar a resposta, devolver pro cliente. Funciona pra um protótipo de 20 linhas, mas cresce mal:

```go
// versão ingênua: handler HTTP, chamada ao provedor de LLM e regra de negócio, tudo misturado
func chatHandler(w http.ResponseWriter, r *http.Request) {
    var req struct{ Prompt string }
    json.NewDecoder(r.Body).Decode(&req)

    body, _ := json.Marshal(map[string]string{"prompt": req.Prompt})
    httpReq, _ := http.NewRequest(http.MethodPost, "https://api.anthropic.com/v1/messages", bytes.NewReader(body))
    httpReq.Header.Set("Authorization", "Bearer "+os.Getenv("LLM_API_KEY")) // lido direto aqui, sem validar antes
    resp, _ := http.DefaultClient.Do(httpReq)                              // sem timeout, sem tratar erro
    defer resp.Body.Close()

    var result struct{ Text string }
    json.NewDecoder(resp.Body).Decode(&result)
    json.NewEncoder(w).Encode(map[string]string{"text": result.Text})
}
```

Além de ignorar vários erros silenciosamente (`_` em todo lugar onde deveria ter tratamento — o oposto do que [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) ensina), esse handler é praticamente impossível de testar sem realmente chamar a API da Anthropic pela rede — não existe nenhum ponto onde trocar "a chamada real pro provedor" por "uma versão falsa, controlada pelo teste". A saída é separar responsabilidades em camadas (ver [Clean Architecture](../clean-architecture-separacao-em-camadas/)), cada uma no seu próprio pacote.

## Esboço de arquitetura

```
llmproxy/
  main.go               // monta tudo, sobe o servidor (única camada que conhece TODAS as outras)
  domain/
    chat.go               // ChatRequest, ChatResponse — tipos de domínio, sem saber de HTTP nem do provedor de LLM
  llm/
    client.go              // interface LLMClient — abstração do provedor, sem saber qual provedor de verdade é
  infra/
    anthropic/
      client.go              // implementa LLMClient chamando a API real da Anthropic
  http/
    chat_handler.go           // recebe POST /chat, valida, chama o use case, responde
```

A regra que organiza essas pastas é sempre a mesma: uma camada mais interna (`domain`, `llm`) nunca importa uma camada mais externa (`infra`, `http`) — só o contrário. `domain/chat.go` não tem, em lugar nenhum, um `import "net/http"` nem um `import "infra/anthropic"`; é só `grep import` pra confirmar isso, sem precisar rodar nada (ver [Clean Architecture — o teste rápido](../clean-architecture-separacao-em-camadas/#o-teste-rápido-aplicado)).

## A interface, perto de quem consome

```go
// llm/client.go — o domínio (e o handler HTTP) só conhecem esta interface, nunca o provedor concreto
package llm

import "context"

type LLMClient interface {
    Complete(ctx context.Context, prompt string) (string, error)
}
```

Uma interface de um método só — Interface Segregation Principle na prática (ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md#i--interface-segregation-principle)): quem depende de `LLMClient` só enxerga "pedir uma resposta pra um prompt", nada além disso.

## A implementação real, isolada na camada de infra

```go
// infra/anthropic/client.go — só aqui entra a chamada HTTP real pro provedor
package anthropic

type Client struct {
    apiKey string
    http   *http.Client
}

func NewClient(apiKey string) *Client {
    return &Client{apiKey: apiKey, http: &http.Client{}}
}

func (c *Client) Complete(ctx context.Context, prompt string) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, apiURL, buildBody(prompt))
    if err != nil {
        return "", fmt.Errorf("montando requisição pro LLM: %w", err)
    }
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

`*anthropic.Client` satisfaz `llm.LLMClient` implicitamente (ver [Structs, métodos e interfaces](../structs-metodos-e-interfaces/)) só por ter um método `Complete` com a assinatura certa — não existe declaração `implements LLMClient` em lugar nenhum, o compilador confirma isso sozinho no ponto onde o tipo é usado como a interface.

## O handler HTTP, sem se misturar com a chamada ao provedor

```go
// http/chat_handler.go — camada de entrega: HTTP não se mistura com a lógica de chamar o LLM
func chatHandler(client llm.LLMClient) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req domain.ChatRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "corpo inválido", http.StatusBadRequest)
            return
        }
        text, err := client.Complete(r.Context(), req.Prompt)
        if err != nil {
            writeError(w, err) // ver Tratamento de erros e exceções em APIs
            return
        }
        json.NewEncoder(w).Encode(domain.ChatResponse{Text: text})
    }
}
```

`chatHandler` recebe um `llm.LLMClient` (a interface, não `*anthropic.Client`) como parâmetro, e devolve um `http.HandlerFunc` — uma função que fecha sobre esse `client` (uma **closure**: a função devolvida "lembra" do valor de `client` que recebeu, mesmo depois que `chatHandler` já retornou). Isso é o que torna o handler testável sem chamar o provedor de LLM de verdade.

## Injeção de dependência sem framework, montada no `main.go`

Nenhuma das peças acima decide sozinha qual implementação concreta usar — isso é decidido num único lugar, o `main`, o único ponto do programa que conhece todas as camadas ao mesmo tempo:

```go
// main.go — o único arquivo que conhece TODAS as camadas: domain, llm, infra e http
func main() {
    cfg, err := LoadConfig() // ver Produção — logging, config, error wrapping
    if err != nil {
        log.Fatal(err)
    }

    llmClient := anthropic.NewClient(cfg.LLMAPIKey) // decide a implementação concreta AQUI, e só aqui

    mux := http.NewServeMux()
    mux.HandleFunc("POST /chat", chatHandler(llmClient))

    log.Println("subindo em :" + cfg.Port)
    http.ListenAndServe(":"+cfg.Port, mux)
}
```

Não existe nenhum framework de injeção de dependência envolvido — "injetar" aqui significa só passar `llmClient` como argumento pra `chatHandler`, um construtor comum. Esse é o padrão idiomático de Go pra montar dependências: manual, explícito, visível lendo o `main` de cima a baixo (ver [SOLID em Go — erros comuns de quem está começando](../solid-em-go/#erros-comuns-de-quem-está-começando-em-go)).

## Testável com um `FakeLLMClient`, sem chamar o provedor de verdade

```go
// http/chat_handler_test.go
type FakeLLMClient struct {
    response string
    err      error
}

func (f *FakeLLMClient) Complete(ctx context.Context, prompt string) (string, error) {
    return f.response, f.err
}

func TestChatHandler(t *testing.T) {
    fake := &FakeLLMClient{response: "resposta simulada"}
    handler := chatHandler(fake)

    body := strings.NewReader(`{"prompt":"olá"}`)
    req := httptest.NewRequest(http.MethodPost, "/chat", body)
    w := httptest.NewRecorder()

    handler(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("status = %d, want %d", w.Code, http.StatusOK)
    }
    if !strings.Contains(w.Body.String(), "resposta simulada") {
        t.Errorf("corpo não contém a resposta esperada: %s", w.Body.String())
    }
}
```

`FakeLLMClient` satisfaz a mesma interface `llm.LLMClient` que `*anthropic.Client` satisfaz — o teste troca a implementação real pela fake sem tocar em `chatHandler`, e sem nenhuma chamada de rede real acontecendo durante o teste (ver [Testes (unitários e integração)](../testes-unitarios-e-integracao/)).

## O que isso amarra, do plano inteiro

- `context.WithTimeout` — [Acesso a dados e context.Context](../acesso-a-dados-e-context/): nunca deixar a chamada externa travar pra sempre.
- Interface `LLMClient` declarada perto de quem usa, implementação concreta separada — [SOLID em Go](../solid-em-go/) (DIP) + [Clean Architecture](../clean-architecture-separacao-em-camadas/).
- Erro de rede/provedor virando resposta HTTP consistente — [Tratamento de erros e exceções em APIs](../tratamento-de-erros-e-excecoes-em-apis/).
- Chave de API via variável de ambiente, validada uma vez no boot, nunca hardcoded — [Produção — logging, config, error wrapping](../producao-logging-config-error-wrapping/) e [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#configuração).
- Testável com um `FakeLLMClient` no lugar do `anthropic.Client` real, sem chamada de rede — [Testes](../testes-unitarios-e-integracao/).

Extensão natural se quiser ir além: um worker pool pra processar N prompts em paralelo respeitando o limite de taxa do provedor (ver [Padrões de concorrência para infra](../padroes-de-concorrencia-para-infra/)).

Sem exercício pré-gerado — implementação real fica pra quando você rodar `/exercise projeto final` ou pedir direto no chat.
