# Tratamento de erros e exceções em APIs

Retoma [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) (Módulo 1) aplicado especificamente à borda HTTP — como erro de domínio vira resposta consistente, sem vazar detalhe interno (ver [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#tratamento-de-erro-e-status-code)).

## Erro de domínio como sentinel/tipo, mapeado no handler

```go
// domain
var ErrNotFound = errors.New("não encontrado")
var ErrInvalidAmount = errors.New("valor inválido")

// handler HTTP
func getOrder(w http.ResponseWriter, r *http.Request) {
    order, err := service.FindOrder(r.Context(), r.PathValue("id"))
    if err != nil {
        writeError(w, err)
        return
    }
    json.NewEncoder(w).Encode(order)
}

func writeError(w http.ResponseWriter, err error) {
    status := http.StatusInternalServerError
    switch {
    case errors.Is(err, domain.ErrNotFound):
        status = http.StatusNotFound
    case errors.Is(err, domain.ErrInvalidAmount):
        status = http.StatusBadRequest
    }
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
    // erro interno genérico (status 500): não devolver err.Error() ao cliente — logar detalhado, responder mensagem enxuta
}
```

O domínio não sabe que existe HTTP (ver Clean Architecture) — quem traduz erro de domínio pra status code é sempre a camada de entrega, nunca o contrário.

## Comparando com FastAPI

FastAPI tem `@app.exception_handler(CustomError)` — um decorator central que intercepta exceção e vira resposta. Go não tem exceção subindo sozinha, então o equivalente é a função `writeError` acima chamada explicitamente em cada handler (ou centralizada num middleware, ver [Middleware e chain de handlers](../middleware-e-chain-de-handlers/)) — mais repetitivo à primeira vista, mas o caminho do erro é sempre visível no código, nunca implícito.

## Erro de validação vs erro de negócio vs erro de infra

Três categorias que merecem status code diferente e não devem se misturar num `if` só: validação de input (400, ver [Validação de input](../validacao-de-input/)), regra de negócio violada (400/409, é esperado, o domínio sabe nomear), falha de infra (500, banco caiu, timeout — não é "culpa" do cliente, e o log interno deve ter todo o detalhe que a resposta não tem).
