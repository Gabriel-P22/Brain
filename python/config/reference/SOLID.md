# SOLID em Python — exemplos

Só exemplos de código — serve como âncora de comparação (linguagem que o usuário já domina) ao estudar como o mesmo princípio muda de forma em Go. Definição de cada princípio está em [contexts/common/SOLID.md](../../../contexts/common/SOLID.md) — leia lá primeiro.

Modelo de Python: herança de classe disponível (mas composição geralmente preferível), duck typing, `ABC`/`Protocol` pra contrato explícito quando necessário.

## S — Single Responsibility

```python
# Ruim: Order sabe validar, persistir e notificar.
class Order:
    def validate(self): ...
    def save(self, db): ...
    def notify_customer(self): ...

# Melhor: cada responsabilidade em seu lugar.
class Order:
    def validate(self): ...

class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...

class Notifier(Protocol):
    def notify_order(self, order: Order) -> None: ...
```

## O — Open/Closed

```python
# Ruim: cada novo método de pagamento exige editar process().
def process(kind: str, amount: float) -> None:
    if kind == "pix": ...
    elif kind == "boleto": ...

# Melhor: novo método de pagamento = nova classe, process não muda.
class PaymentMethod(Protocol):
    def pay(self, amount: float) -> None: ...

def process(method: PaymentMethod, amount: float) -> None:
    method.pay(amount)
```

## L — Liskov Substitution

```python
# Ruim: sobrescreve o método restringindo o que a classe base aceitava.
class Notifier:
    def send(self, message: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        if len(message) > 100:
            raise ValueError("mensagem muito longa")  # Notifier base não tinha essa restrição
        ...

# Melhor: contrato da base é respeitado, restrição vira validação anterior (fora da hierarquia).
class Notifier(Protocol):
    def send(self, message: str) -> None: ...

class EmailNotifier:
    def send(self, message: str) -> None: ...
```

## I — Interface Segregation

```python
# Ruim: um Protocol "canivete suíço", a maioria dos consumidores usa só uma fração.
class Storage(Protocol):
    def save(self, key: str, data: bytes) -> None: ...
    def load(self, key: str) -> bytes: ...
    def delete(self, key: str) -> None: ...
    def list(self) -> list[str]: ...
    def backup(self) -> None: ...

# Melhor: protocolos pequenos, cada consumidor pede só o que usa.
class Saver(Protocol):
    def save(self, key: str, data: bytes) -> None: ...

class Loader(Protocol):
    def load(self, key: str) -> bytes: ...
```

## D — Dependency Inversion

```python
# domain/order.py — a abstração vive aqui, perto de quem usa.
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...

# infra/postgres_repo.py — implementação concreta, não conhecida pelo domain.
class PostgresOrderRepo:
    def __init__(self, conn): self.conn = conn
    def save(self, order: Order) -> None: ...
```

Injeção costuma ser manual (passar a dependência no `__init__`) ou via framework (ex: `Depends` do FastAPI) — a abstração (`Protocol`) nunca deve depender do módulo de infra.
