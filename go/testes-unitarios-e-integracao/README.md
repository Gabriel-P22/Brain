# Testes (unitários e integração)

Table-driven test básico já foi visto em [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/) (Módulo 1). Aqui: aplicar em handler HTTP e em teste de integração real.

## Testando handler HTTP sem subir servidor

```go
func TestGetOrderHandler(t *testing.T) {
    repo := NewFakeOrderRepo()   // ver Repository pattern na prática
    repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    service := NewOrderService(repo)

    req := httptest.NewRequest(http.MethodGet, "/orders/1", nil)
    req.SetPathValue("id", "1")
    w := httptest.NewRecorder()

    getOrderHandler(service)(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("status = %d, want %d", w.Code, http.StatusOK)
    }
}
```

`httptest.NewRequest`/`NewRecorder` simulam request/response sem porta de rede real aberta — equivalente ao `TestClient` do FastAPI ou `test_client` do Flask, só que stdlib.

## Teste unitário com fake, teste de integração com banco real

Unitário (rápido, roda sempre): usa `FakeOrderRepo` in-memory — testa a lógica do `OrderService` isolada.

Integração (mais lento, roda menos vezes — CI, não a cada save): usa `*postgres.OrderRepo` de verdade, contra um banco real ou containerizado:

```go
func TestOrderRepo_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("pulando teste de integração")   // `go test -short` pula
    }
    db := setupTestDB(t)   // sobe um Postgres via testcontainers-go, ou aponta pro banco de CI
    repo := postgres.NewOrderRepo(db)

    err := repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    // ...
}
```

`testcontainers-go` sobe um container Docker (Postgres real) só pra duração do teste — equivalente ao que `pytest` + `testcontainers-python` ou uma fixture de banco efêmero fariam em Python.

## Por que a dupla implementação (fake + real) compensa

Isso só é barato de manter porque a interface `OrderRepository` (ver [DDD em Go](../ddd-em-go/)) é pequena e estável — trocar implementação sem editar o consumidor é literalmente Open/Closed + Dependency Inversion na prática de teste, não só em produção.

## go test -race

Sempre que o tópico envolver concorrência (worker pool, cache compartilhado), rodar `go test -race` — pega condição de corrida que passaria despercebida num teste sequencial normal.
