# Validação de input

## Onde a validação mora — Clean Architecture aplicada

Validação de formato/shape (campo obrigatório, tamanho, formato de e-mail) é preocupação da camada de entrega, não do domínio — o domínio valida regra de negócio (`Order.Cancel()` recusar cancelar pedido já enviado), a borda valida que o request faz sentido antes disso. Ver [Clean Architecture](../clean-architecture-separacao-em-camadas/).

## Validação manual (stdlib)

```go
type CreateOrderRequest struct {
    Amount float64 `json:"amount"`
    Email  string  `json:"email"`
}

func (r CreateOrderRequest) Validate() error {
    if r.Amount <= 0 {
        return errors.New("amount deve ser positivo")
    }
    if !strings.Contains(r.Email, "@") {
        return errors.New("email inválido")
    }
    return nil
}
```

Sem decorator/anotação — `Validate()` é só um método comum, chamado explicitamente no handler antes de seguir. Verboso comparado a Pydantic, mas nada acontece "magicamente" fora do fluxo visível de código.

## Validação declarativa via tag (ecossistema Gin)

```go
type CreateOrderRequest struct {
    Amount float64 `json:"amount" binding:"required,gt=0"`
    Email  string  `json:"email" binding:"required,email"`
}

// no handler Gin:
var req CreateOrderRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}
```

`go-playground/validator` (usado por baixo do `binding` do Gin) é o mais próximo de Pydantic que existe no ecossistema Go — tag declarativa, mensagem de erro gerada automaticamente. Ainda assim, menos automático que Pydantic: não gera schema JSON/OpenAPI sozinho, não faz coerção de tipo tão flexível.

## Erro de validação é 400, sempre

Ver [Tratamento de erros e exceções em APIs](../tratamento-de-erros-e-excecoes-em-apis/) — erro de validação nunca deveria virar 500. Se a mensagem de validação pode ir direto pro cliente (diferente de erro de infra), ela é informação útil, não vazamento de detalhe interno.
