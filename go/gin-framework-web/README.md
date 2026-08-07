# Gin (framework web)

## Quando sair da stdlib

`net/http` (com o `ServeMux` 1.22+) resolve roteamento básico. Gin compensa quando você quer, de fábrica: binding de JSON pro struct com validação (via tag), ecossistema grande de middleware pronto, e performance de radix tree no router. Comparável a "por que usar FastAPI/Flask em vez de `http.server` puro" em Python — mesma motivação, framework tira boilerplate repetitivo.

## API básica

```go
r := gin.Default()   // já vem com logger + recovery middleware

r.GET("/orders/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{"id": id})
})

r.POST("/orders", func(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // ...
})

r.Run(":8080")
```

`c.ShouldBindJSON` já decodifica e valida em um passo (via tag `binding:"required"` no struct) — mais próximo da experiência de Pydantic/FastAPI do que a stdlib pura, que exige decode manual + validação manual separada (ver [Validação de input](../validacao-de-input/)).

## gin.Context como parâmetro único

Diferente da stdlib (`ResponseWriter` + `*Request` separados), Gin embrulha os dois num `*gin.Context` só, que também carrega o middleware chain (`c.Next()`) — ver [Middleware e chain de handlers](../middleware-e-chain-de-handlers/) pra aprofundar nisso.

## Rota agrupada

```go
api := r.Group("/v1")
api.Use(authMiddleware)
{
    api.GET("/orders", listOrders)
    api.POST("/orders", createOrder)
}
```

Grupo com prefixo + middleware compartilhado é o equivalente do `APIRouter` do FastAPI ou `Blueprint` do Flask.
