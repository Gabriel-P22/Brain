# Repository pattern na prática

Retomando [DDD em Go](../ddd-em-go/) e [Clean Architecture](../clean-architecture-separacao-em-camadas/) — aqui é o exemplo completo, com duas implementações da mesma interface (o ponto principal do padrão: trocar implementação sem tocar em quem consome).

## A interface, no domínio

```go
// domain/order.go
package domain

type OrderRepository interface {
    Save(ctx context.Context, o Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}
```

## Implementação real, na infra

```go
// infra/postgres/order_repo.go
package postgres

type OrderRepo struct{ db *sql.DB }

func NewOrderRepo(db *sql.DB) *OrderRepo { return &OrderRepo{db: db} }

func (r *OrderRepo) Save(ctx context.Context, o domain.Order) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO orders (id, amount) VALUES ($1, $2) ON CONFLICT (id) DO UPDATE SET amount = $2",
        o.ID, o.Amount)
    return err
}

func (r *OrderRepo) FindByID(ctx context.Context, id string) (*domain.Order, error) {
    var o domain.Order
    err := r.db.QueryRowContext(ctx, "SELECT id, amount FROM orders WHERE id = $1", id).
        Scan(&o.ID, &o.Amount)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("pedido %s: %w", id, ErrNotFound)
    }
    return &o, err
}
```

## Implementação fake, pra teste

```go
// domain/order_repo_fake.go (ou junto do arquivo de teste)
type FakeOrderRepo struct {
    data map[string]domain.Order
}

func NewFakeOrderRepo() *FakeOrderRepo {
    return &FakeOrderRepo{data: make(map[string]domain.Order)}
}

func (r *FakeOrderRepo) Save(ctx context.Context, o domain.Order) error {
    r.data[o.ID] = o
    return nil
}

func (r *FakeOrderRepo) FindByID(ctx context.Context, id string) (*domain.Order, error) {
    o, ok := r.data[id]
    if !ok {
        return nil, ErrNotFound
    }
    return &o, nil
}
```

`OrderService` (a camada de aplicação) recebe `OrderRepository` no construtor — em produção, `main.go` injeta `*postgres.OrderRepo`; em teste, o próprio teste injeta `*FakeOrderRepo`, sem precisar de banco real nem mock framework (`unittest.mock` do Python tem uso parecido, mas aqui é só um struct comum implementando a mesma interface — sem lib nenhuma). Isso é o "dá pra testar isolado?" do Clean Architecture, na prática.
