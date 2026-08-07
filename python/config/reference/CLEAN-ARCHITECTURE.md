# Clean Architecture em Python — exemplo

Só exemplo de estrutura. Definição de cada camada está em [contexts/common/CLEAN-ARCHITECTURE.md](../../../contexts/common/CLEAN-ARCHITECTURE.md) — leia lá primeiro.

```
order/
  domain.py          # Order, OrderStatus, regra de negócio pura — zero import de infra
  repository.py        # Protocol OrderRepository — pertence ao domínio, não à infra
  service.py             # use case: orquestra domain + repository, ainda sem saber QUAL banco
infra/postgres/
  order_repo.py           # implementa OrderRepository — só aqui entra psycopg/SQLAlchemy
infra/http/
  order_router.py           # use case chamado a partir de uma request — só aqui entra FastAPI/Flask
```

```python
# service.py — camada de aplicação/caso de uso
class OrderService:
    def __init__(self, repo: OrderRepository):   # depende do Protocol, não da classe concreta
        self.repo = repo

    def cancel(self, id: str) -> None:
        order = self.repo.find_by_id(id)
        order.cancel()          # regra de negócio vive na entidade, não aqui
        self.repo.save(order)
```

`domain.py` não importa driver de banco nem framework web — teste de `Order.cancel()` roda sem banco nem servidor no ar. O ponto de composição (`OrderService(PostgresOrderRepo(conn))`) fica isolado — em FastAPI, tipicamente resolvido via `Depends`.
