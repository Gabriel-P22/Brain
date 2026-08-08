# Tratamento de erros e exceções em APIs

Retoma [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) (onde `error`, `errors.Is`/`errors.As` e wrapping com `%w` foram explicados do zero) e [net/http e APIs REST](../net-http-e-apis-rest/) (onde `http.ResponseWriter` e status code foram vistos). Aqui: como um erro que aconteceu lá dentro da regra de negócio vira uma resposta HTTP consistente, sem vazar detalhe interno pro cliente (ver [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#tratamento-de-erro-e-status-code)).

## O problema: quem decide o status code?

Uma API HTTP sempre devolve um **código de status** — um número de 3 dígitos que diz ao cliente o que aconteceu com o pedido dele (`200` deu certo, `404` não achou o recurso, `400` o pedido estava mal formado, `500` algo quebrou do lado do servidor). Quem gera o erro (uma função lá no meio da regra de negócio, que nem sabe que existe HTTP por trás) não deveria decidir esse número — só quem está na borda, lidando com a requisição HTTP, sabe traduzir "pedido não encontrado" para "404".

Isso é literalmente a regra de dependência de Clean Architecture na prática (ver [Clean Architecture](../clean-architecture-separacao-em-camadas/)): o domínio não importa `net/http`, então ele não pode devolver um `http.StatusNotFound` diretamente — ele devolve um erro que descreve *o que* aconteceu, e é a camada de entrega (o handler) quem traduz isso pra *como* isso aparece pro cliente.

## Erro de domínio como sentinel, mapeado no handler

```go
// domain/errors.go — o domínio só descreve O QUE aconteceu, sem saber de HTTP
package domain

import "errors"

var ErrNotFound = errors.New("não encontrado")
var ErrInvalidAmount = errors.New("valor inválido")
```

```go
// http/order_handler.go — só aqui, na borda, o erro vira status code
package http

func getOrder(w http.ResponseWriter, r *http.Request) {
    order, err := service.FindOrder(r.Context(), r.PathValue("id"))
    if err != nil {
        writeError(w, err)
        return
    }
    json.NewEncoder(w).Encode(order)
}

func writeError(w http.ResponseWriter, err error) {
    status := http.StatusInternalServerError // padrão: "algo deu errado, e não é culpa do cliente"
    switch {
    case errors.Is(err, domain.ErrNotFound):
        status = http.StatusNotFound
    case errors.Is(err, domain.ErrInvalidAmount):
        status = http.StatusBadRequest
    }
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
    // status 500 (o caso default): nunca devolver err.Error() cru pro cliente aqui —
    // logar detalhado no servidor, responder algo genérico ("erro interno") pro cliente
}
```

`errors.Is` compara o erro recebido (que pode estar embrulhado em várias camadas de `fmt.Errorf("...: %w", err)`) contra um valor sentinela específico — é assim que `writeError` sabe "esse erro, não importa de onde veio na cadeia de chamadas, representa um recurso não encontrado".

## Por que centralizar isso numa função, em vez de repetir em cada handler

Uma primeira versão "ingênua", sem pensar em reuso, ficaria assim:

```go
// versão ingênua: cada handler decide o status code sozinho, duplicando a lógica
func getOrder(w http.ResponseWriter, r *http.Request) {
    order, err := service.FindOrder(r.Context(), r.PathValue("id"))
    if err != nil {
        if errors.Is(err, domain.ErrNotFound) {
            w.WriteHeader(http.StatusNotFound)
        } else if errors.Is(err, domain.ErrInvalidAmount) {
            w.WriteHeader(http.StatusBadRequest)
        } else {
            w.WriteHeader(http.StatusInternalServerError)
        }
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }
    json.NewEncoder(w).Encode(order)
}

func cancelOrder(w http.ResponseWriter, r *http.Request) {
    err := service.Cancel(r.Context(), r.PathValue("id"))
    if err != nil {
        // o MESMO bloco if/else de cima, copiado e colado aqui de novo
        if errors.Is(err, domain.ErrNotFound) {
            w.WriteHeader(http.StatusNotFound)
        } else {
            w.WriteHeader(http.StatusInternalServerError)
        }
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

O problema aqui não é só "mais linhas de código" — é que, no dia em que um erro novo precisar de tratamento especial (por exemplo, `ErrDuplicateEmail` virando `409 Conflict`), você precisa lembrar de editar esse bloco `if/else` em *todo* handler que existe, um por um. Esquecer um é fácil e o compilador não avisa. Extrair a tradução "erro → status" pra uma única função (`writeError`, na primeira versão lá em cima) resolve isso: existe um único lugar que sabe mapear erro de domínio pra status HTTP, e todo handler só chama essa função. Isso é Single Responsibility Principle aplicado na prática — a responsabilidade de "traduzir erro pra HTTP" mora num lugar só (ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md#s--single-responsibility-principle)) — e também abre caminho pra fazer isso de forma ainda mais automática, centralizando num middleware (ver [Middleware e chain de handlers](../middleware-e-chain-de-handlers/)), em vez de cada handler chamar `writeError` manualmente.

## Corpo de erro estruturado, não um mapa solto

O exemplo acima usa `map[string]string{"error": err.Error()}` — funciona, mas não deixa claro pra quem lê o código (nem pra quem consome a API, olhando a documentação) qual é exatamente o formato da resposta de erro. Um `struct` nomeado deixa isso explícito e consistente em toda a API, como pede [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#tratamento-de-erro-e-status-code) ("mesmo shape de JSON pra todo erro"):

```go
// http/errors.go
type ErrorResponse struct {
    Code    string `json:"code"`    // identificador estável, pensado pra código consumir (ex: "not_found")
    Message string `json:"message"` // texto pensado pra humano ler
}

func writeError(w http.ResponseWriter, err error) {
    var status int
    var code string

    switch {
    case errors.Is(err, domain.ErrNotFound):
        status, code = http.StatusNotFound, "not_found"
    case errors.Is(err, domain.ErrInvalidAmount):
        status, code = http.StatusBadRequest, "invalid_amount"
    default:
        status, code = http.StatusInternalServerError, "internal_error"
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{
        Code:    code,
        Message: err.Error(),
    })
}
```

Repare que, no caso `default` (erro de infra desconhecido), a mensagem que vai pro `Message` ainda é `err.Error()` — em código de produção real, esse caso normalmente troca por uma mensagem genérica fixa (`"erro interno, tente novamente"`), porque `err.Error()` de uma falha de infra pode conter detalhe sensível (string de conexão, caminho de arquivo). O detalhe completo vai pro log do servidor (ver [Produção — logging, config, error wrapping](../producao-logging-config-error-wrapping/)), nunca pra resposta.

## Três categorias de erro, e por que não misturar num `if` só

Nem todo erro é igual, e cada categoria merece tratamento diferente:

1. **Erro de validação de input** — o formato do pedido está errado (campo obrigatório faltando, e-mail sem `@`). Sempre `400 Bad Request`. É "culpa" do cliente, no sentido de que ele pode corrigir e tentar de novo. Ver [Validação de input](../validacao-de-input/).
2. **Erro de regra de negócio violada** — o pedido está bem formado, mas a operação não é permitida naquele estado (`Order.Cancel()` recusando cancelar um pedido que já foi enviado). Normalmente `400` ou `409 Conflict`. O domínio sabe nomear esse erro (`ErrOrderAlreadyShipped`), porque é regra dele.
3. **Erro de infra** — o banco caiu, a rede deu timeout, um serviço externo não respondeu. Sempre `500 Internal Server Error` (ou, se for especificamente "dependência externa fora do ar", às vezes `503 Service Unavailable`). Não é "culpa" do cliente — ele não tem como corrigir só tentando de novo com dados diferentes.

```go
func classify(err error) string {
    switch {
    case errors.Is(err, domain.ErrInvalidAmount):
        return "validação: cliente pode corrigir e reenviar"
    case errors.Is(err, domain.ErrOrderAlreadyShipped):
        return "regra de negócio: operação não permitida neste estado"
    case errors.Is(err, context.DeadlineExceeded):
        return "infra: timeout — não é culpa do cliente"
    default:
        return "desconhecido: tratar como infra (500) por segurança"
    }
}
```

Misturar essas três categorias num único `if`/`switch` sem essa separação mental é a origem mais comum de bug de status code errado em API — devolver `500` pra um erro de validação (assusta o cliente e polui alerta de monitoramento com "erro" que não é erro de verdade), ou devolver `400` pra uma falha de banco (engana o cliente, fazendo ele achar que o problema é dele).
