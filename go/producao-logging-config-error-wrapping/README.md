# Produção — logging, config, error wrapping

## Logging estruturado — log/slog é stdlib (desde Go 1.21)

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("pedido criado", "order_id", order.ID, "amount", order.Amount)
// {"time":"...","level":"INFO","msg":"pedido criado","order_id":"123","amount":42.5}
```

Antes do 1.21, log estruturado em Go dependia de lib de terceiro (`zap`, `zerolog`, `logrus`) — hoje a stdlib cobre o caso comum. Comparável ao `structlog`/`logging` com `JSONRenderer` do Python, mas nativo, sem dependência.

## Correlation ID por requisição

```go
func withRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := uuid.NewString()
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        logger.InfoContext(ctx, "request iniciado", "request_id", id, "path", r.URL.Path)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

`context.WithValue` é o único uso idiomático aceito de "carregar dado no context" (fora cancelamento/timeout) — carregar request ID, não dado de negócio (isso vira parâmetro explícito). Ver [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#observabilidade) pra por que isso importa: sem ID de correlação, depurar concorrência em produção é adivinhação.

## Configuração via variável de ambiente

```go
type Config struct {
    DatabaseURL string
    Port        string
    LLMAPIKey   string
}

func LoadConfig() (Config, error) {
    key := os.Getenv("LLM_API_KEY")
    if key == "" {
        return Config{}, errors.New("LLM_API_KEY não definida")
    }
    return Config{
        DatabaseURL: os.Getenv("DATABASE_URL"),
        Port:        cmp.Or(os.Getenv("PORT"), "8080"),   // cmp.Or (Go 1.22+): primeiro não-zero
        LLMAPIKey:   key,
    }, nil
}
```

Falhar cedo (no boot, não na primeira request que precisar da config ausente) é o padrão — `LoadConfig` roda uma vez no `main`, antes de subir o servidor. Sem `.env` carregado magicamente como o `python-dotenv`/`pydantic-settings` fazem — em Go, ou a variável já está no ambiente (export manual, Docker `ENV`, etc.) ou você lê um `.env` explicitamente com uma lib pequena (`godotenv`) só em dev.

## Error wrapping, revisitado com contexto de produção

Já visto em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) — em produção, a cadeia de `%w` é o que aparece no log quando algo falha 3 camadas abaixo de onde o erro é logado. Logar só `err.Error()` sem ter encadeado contexto no caminho é a diferença entre um log útil ("carregando usuário 123: conectando postgres: timeout") e um log inútil ("timeout").
