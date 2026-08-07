# Clean Architecture em Go — exemplo

Só exemplo de estrutura. Definição de cada camada está em [contexts/common/CLEAN-ARCHITECTURE.md](../../../contexts/common/CLEAN-ARCHITECTURE.md) — leia lá primeiro.

```
order/
  domain.go         // Order, OrderStatus, regra de negócio pura — zero import de infra
  repository.go       // interface OrderRepository — pertence ao domínio, não à infra
  service.go            // use case: orquestra domain + repository, ainda sem saber QUAL banco
infra/postgres/
  order_repo.go          // implementa order.OrderRepository — só aqui entra database/sql
infra/http/
  order_handler.go         // use case chamado a partir de uma request HTTP — só aqui entra net/http
```

```go
// service.go — camada de aplicação/caso de uso
type OrderService struct {
    repo OrderRepository   // depende da interface do domínio, não de *postgres.OrderRepo
}

func NewOrderService(repo OrderRepository) *OrderService {
    return &OrderService{repo: repo}
}

func (s *OrderService) Cancel(id string) error {
    o, err := s.repo.FindByID(id)
    if err != nil {
        return err
    }
    if err := o.Cancel(); err != nil {   // regra de negócio vive na entidade, não aqui
        return err
    }
    return s.repo.Save(*o)
}
```

`domain.go` não importa `database/sql` nem `net/http` — teste de `Order.Cancel()` roda sem banco nem servidor no ar. `main.go` é quem monta tudo (`NewOrderService(postgres.NewOrderRepo(db))`), único lugar que conhece a implementação concreta.
