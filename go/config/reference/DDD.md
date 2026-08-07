# DDD em Go — exemplos

Só exemplos de código. Definição de cada bloco está em [contexts/common/DDD.md](../../../contexts/common/DDD.md) — leia lá primeiro.

## Entidade

```go
type Order struct {
    ID     string
    Amount float64
    Status OrderStatus
}

func (o *Order) Cancel() error {
    if o.Status == StatusShipped {
        return errors.New("pedido já enviado, não pode cancelar")
    }
    o.Status = StatusCancelled
    return nil
}
```

## Value Object

```go
type Money struct {
    Amount   int64  // centavos, evita ponto flutuante em dinheiro
    Currency string
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, errors.New("moedas diferentes")
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}
```

## Agregado

```go
type Order struct {   // Order é a raiz do agregado
    ID    string
    Items []OrderItem  // OrderItem só existe dentro de um Order — sem repositório próprio
}

func (o *Order) AddItem(item OrderItem) error {
    if len(o.Items) >= 50 {
        return errors.New("limite de itens excedido")
    }
    o.Items = append(o.Items, item)
    return nil
}
```

## Repository

```go
type OrderRepository interface {
    Save(Order) error
    FindByID(id string) (*Order, error)
}
```

## Domain Service

```go
func TransferFunds(from, to *Account, amount Money) error {
    if err := from.Withdraw(amount); err != nil {
        return err
    }
    return to.Deposit(amount)
}
```

## Bounded Context

Em Go, bounded context tende a virar módulo/pacote de alto nível (`billing/`, `shipping/`) — cada um com seu próprio `Order` se o significado for diferente em cada contexto, mesmo com nome igual.
