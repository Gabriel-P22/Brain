# SOLID em Go — exemplos

Só exemplos de código. Definição de cada princípio está em [contexts/common/SOLID.md](../../../contexts/common/SOLID.md) — leia lá primeiro.

Modelo de Go: sem herança de classe — composição, interfaces implícitas (satisfeitas automaticamente, sem `implements`) e, por convenção, interfaces pequenas (1-2 métodos) definidas do lado de quem consome, não de quem implementa.

## S — Single Responsibility

```go
// Ruim: Order sabe validar, persistir e notificar.
type Order struct{ /* ... */ }
func (o *Order) Validate() error { /* ... */ }
func (o *Order) Save(db *sql.DB) error { /* ... */ }
func (o *Order) NotifyCustomer() error { /* ... */ }

// Melhor: cada responsabilidade em seu lugar.
type Order struct{ /* ... */ }
func (o Order) Validate() error { /* ... */ }

type OrderRepository interface{ Save(Order) error }
type Notifier interface{ NotifyOrder(Order) error }
```

## O — Open/Closed

```go
// Ruim: cada novo tipo de pagamento exige editar Process().
func Process(kind string, amount float64) error {
    switch kind {
    case "pix": /* ... */
    case "boleto": /* ... */
    }
    return nil
}

// Melhor: novo método de pagamento = novo tipo, Process não muda.
type PaymentMethod interface{ Pay(amount float64) error }
func Process(m PaymentMethod, amount float64) error { return m.Pay(amount) }
```

## L — Liskov Substitution

```go
// Ruim: satisfaz a assinatura de io.Reader, mas quebra o contrato de comportamento.
type BadReader struct{}
func (BadReader) Read(p []byte) (int, error) {
    return 0, errors.New("acabou") // deveria retornar io.EOF quando o stream termina
}
// Qualquer código que faz `for { n, err := r.Read(buf); if err == io.EOF { break } }`
// quebra com esse Reader — mesmo compilando, ele não é substituível por um io.Reader real.

// Melhor: respeita o contrato implícito da interface, não só a assinatura.
type GoodReader struct{ data []byte; pos int }
func (r *GoodReader) Read(p []byte) (int, error) {
    if r.pos >= len(r.data) {
        return 0, io.EOF
    }
    n := copy(p, r.data[r.pos:])
    r.pos += n
    return n, nil
}
```

## I — Interface Segregation

```go
// Ruim: interface grande, cada consumidor usa só uma fração dela.
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
    Delete(key string) error
    List() ([]string, error)
    Backup() error
}

// Melhor: quebrada em interfaces pequenas — o consumidor pede só o que usa.
type Saver interface{ Save(key string, data []byte) error }
type Loader interface{ Load(key string) ([]byte, error) }
```

## D — Dependency Inversion

```go
// domain/order.go — a abstração vive aqui, perto de quem usa.
package domain
type OrderRepository interface{ Save(Order) error }

// infra/postgres/order_repo.go — implementação concreta, não conhecida pelo domain.
package postgres
type OrderRepo struct{ db *sql.DB }
func (r *OrderRepo) Save(o domain.Order) error { /* ... */ }
```

Sem container de DI por padrão: injeção é manual, via construtor (`NewService(repo Repository)`), montada em `main.go`.
