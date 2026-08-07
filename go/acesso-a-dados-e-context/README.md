# Acesso a dados e context.Context

## database/sql — driver + pool embutido

```go
db, err := sql.Open("postgres", dsn)   // "postgres" é o driver registrado (ex: lib/pq, pgx)
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(5)

row := db.QueryRowContext(ctx, "SELECT amount FROM orders WHERE id = $1", id)
var amount float64
err = row.Scan(&amount)
```

`sql.Open` não conecta de fato (lazy) — a conexão real acontece na primeira query. Pool de conexão já vem embutido na stdlib (diferente de precisar de `SQLAlchemy` + engine configurado à parte em Python) — `SetMaxOpenConns` é o parâmetro mais importante de configurar cedo, senão o default pode não bater com o limite do banco em produção.

`Scan` preenche variável por ponteiro, célula a célula — sem ORM mapeando struct inteiro automaticamente (isso é o que uma lib como `sqlx` ou `gorm` adiciona por cima, opcional).

## context.Context — controle de vida da operação

Todo método de I/O idiomático em Go aceita `ctx context.Context` como primeiro parâmetro — carrega prazo, cancelamento e valores de request através de toda a cadeia de chamada:

```go
func (r *OrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    row := r.db.QueryRowContext(ctx, "...", id)
    // ...
}
```

```go
ctx, cancel := context.WithTimeout(parentCtx, 3*time.Second)
defer cancel()

order, err := repo.FindByID(ctx, id)
if errors.Is(err, context.DeadlineExceeded) {
    // a query não terminou a tempo — banco lento, rede, etc.
}
```

Não tem equivalente direto e onipresente em Python — o mais próximo é passar um timeout manualmente pra cada chamada (`requests.get(url, timeout=5)`) ou usar `asyncio.timeout()` em código assíncrono, mas em Go isso é parte do idiom da linguagem, propagado explicitamente em cada função que faz I/O, não uma feature isolada de uma lib.

## Cancelamento em cascata

Se o `ctx` do request HTTP for cancelado (cliente desconectou, ex: fechou a aba), qualquer `ctx` derivado dele (passado pra query de banco, chamada de API externa) também cancela — a query em andamento é abortada em vez de terminar um trabalho que ninguém mais vai usar. Isso é automático desde que você propague o `ctx` recebido, em vez de trocar por `context.Background()` no meio do caminho (erro comum: "esquecer" de propagar e cortar a cadeia de cancelamento sem querer).
