# DDD em Python — exemplos

Só exemplos de código — âncora de comparação ao estudar Go. Definição de cada bloco está em [contexts/common/DDD.md](../../../contexts/common/DDD.md) — leia lá primeiro.

## Entidade

```python
@dataclass
class Order:
    id: str
    amount: float
    status: OrderStatus

    def cancel(self) -> None:
        if self.status == OrderStatus.SHIPPED:
            raise ValueError("pedido já enviado, não pode cancelar")
        self.status = OrderStatus.CANCELLED
```

## Value Object

```python
@dataclass(frozen=True)  # frozen = imutável, igualdade por valor já vem do dataclass
class Money:
    amount: int   # centavos
    currency: str

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("moedas diferentes")
        return Money(self.amount + other.amount, self.currency)
```

## Agregado

```python
@dataclass
class Order:   # Order é a raiz do agregado
    id: str
    items: list["OrderItem"] = field(default_factory=list)

    def add_item(self, item: "OrderItem") -> None:
        if len(self.items) >= 50:
            raise ValueError("limite de itens excedido")
        self.items.append(item)
```

## Repository

```python
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def find_by_id(self, id: str) -> Order | None: ...
```

## Domain Service

```python
def transfer_funds(from_acc: Account, to_acc: Account, amount: Money) -> None:
    from_acc.withdraw(amount)
    to_acc.deposit(amount)
```

## Bounded Context

Em Python, bounded context costuma virar um pacote/subpacote de alto nível (`billing/`, `shipping/`) — cada um com seu próprio `Order` se o significado for diferente em cada contexto, mesmo com nome igual.
