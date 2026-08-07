# Middleware e chain de handlers

## O padrão na stdlib: func que envolve Handler

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s — %v", r.Method, r.URL.Path, time.Since(start))
    })
}

mux := http.NewServeMux()
mux.HandleFunc("GET /orders/{id}", getOrder)
handler := loggingMiddleware(mux)   // encadeia — pode empilhar mais de um
http.ListenAndServe(":8080", handler)
```

Middleware é literalmente uma função de ordem superior: recebe um `Handler`, devolve outro que faz algo antes/depois de chamar o original. Sem decorator (`@app.middleware("http")` do FastAPI) — é composição de função explícita.

## recover — o uso legítimo de panic/recover

Retomando de [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/): panic não é fluxo normal, mas se acontecer um numa goroutine de request, sem tratamento ela derruba o processo inteiro (todas as outras requisições em andamento também caem). Middleware de recovery existe exatamente pra isso:

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recuperado: %v", err)
                w.WriteHeader(http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

Isso é o que `gin.Default()` já inclui de fábrica (junto com logging) — ver [Gin (framework web)](../gin-framework-web/).

## Chain no Gin

```go
r := gin.New()
r.Use(loggingMiddleware, authMiddleware)   // aplica a TODA rota registrada depois

func authMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatus(http.StatusUnauthorized)   // corta a chain — handler final nunca roda
        return
    }
    c.Next()   // segue pro próximo da chain
}
```

`c.Next()`/`c.Abort()` deixam explícito onde a chain continua ou para — mecanismo equivalente ao `yield`/`await call_next()` do middleware de FastAPI/Starlette, mas sem `async`.

## Ordem importa

Middleware registrado primeiro executa primeiro na entrada e por último na saída (como pilha) — `recovery` deveria vir antes de tudo (pra capturar panic de qualquer middleware seguinte), `auth` antes da lógica de negócio mas depois de log/recovery.
