# Testes (unitários e integração)

O básico de teste em Go (arquivo `xxx_test.go`, função `TestAlgo(t *testing.T)`, table-driven test com `t.Run`) já foi explicado do zero em [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/#testes-em-go). Aqui: aplicar esse mesmo mecanismo a um handler HTTP inteiro, e entender a diferença entre teste unitário e teste de integração.

## Testando um handler HTTP sem abrir porta de rede nenhuma

Testar um handler HTTP de verdade — subindo o servidor, abrindo uma porta TCP, fazendo uma requisição de rede real contra ele — seria lento e frágil (depende de porta livre, de rede funcionando). O pacote `net/http/httptest`, também parte da stdlib, resolve isso simulando a requisição e a resposta inteiramente em memória:

```go
func TestGetOrderHandler(t *testing.T) {
    repo := NewFakeOrderRepo() // ver Repository pattern na prática
    repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    service := NewOrderService(repo)

    req := httptest.NewRequest(http.MethodGet, "/orders/1", nil)
    req.SetPathValue("id", "1") // simula o valor que o ServeMux extrairia de /orders/{id}
    w := httptest.NewRecorder()

    getOrderHandler(service)(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("status = %d, want %d", w.Code, http.StatusOK)
    }
}
```

`httptest.NewRequest` cria um `*http.Request` que se comporta como se tivesse vindo de uma requisição de rede real, mas sem nenhuma rede envolvida. `httptest.NewRecorder` cria um `*httptest.ResponseRecorder` que implementa a mesma interface `http.ResponseWriter` que um handler normalmente escreveria — só que, em vez de mandar bytes pra uma conexão TCP, ele grava tudo em memória (`w.Code`, `w.Body`), pra o teste poder inspecionar depois. O handler não sabe a diferença entre um `*httptest.ResponseRecorder` e um `http.ResponseWriter` de verdade — pra ele, os dois satisfazem a mesma interface (ver [Structs, métodos e interfaces](../structs-metodos-e-interfaces/)), e é exatamente essa substituibilidade que torna esse teste possível sem porta de rede real.

## Tabela de casos aplicada a um handler

O mesmo padrão table-driven já visto em testes de função pura funciona igual aqui, cobrindo vários cenários de requisição numa única função de teste:

```go
func TestGetOrderHandler_Cases(t *testing.T) {
    repo := NewFakeOrderRepo()
    repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    service := NewOrderService(repo)
    handler := getOrderHandler(service)

    cases := []struct {
        name       string
        orderID    string
        wantStatus int
    }{
        {"pedido existente", "1", http.StatusOK},
        {"pedido inexistente", "999", http.StatusNotFound},
        {"id vazio", "", http.StatusBadRequest},
    }

    for _, c := range cases {
        t.Run(c.name, func(t *testing.T) {
            req := httptest.NewRequest(http.MethodGet, "/orders/"+c.orderID, nil)
            req.SetPathValue("id", c.orderID)
            w := httptest.NewRecorder()

            handler(w, req)

            if w.Code != c.wantStatus {
                t.Errorf("status = %d, want %d", w.Code, c.wantStatus)
            }
        })
    }
}
```

Uma única função de teste cobre três cenários diferentes, cada um rodando como um subteste nomeado (`t.Run`) — se o cenário "pedido inexistente" falhar, o relatório de teste mostra exatamente qual `name` falhou, sem precisar adivinhar em qual das três chamadas o problema estava.

## Teste unitário com fake vs. teste de integração com dependência real

**Unitário** (rápido, roda a toda hora — a cada save do arquivo, em CI a cada commit): usa uma implementação **fake** da interface de repositório, guardando dados só em memória (um `map`), sem nenhuma dependência externa:

```go
func TestOrderService_FindOrder(t *testing.T) {
    repo := NewFakeOrderRepo()
    repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    service := NewOrderService(repo)

    order, err := service.FindOrder(context.Background(), "1")
    if err != nil {
        t.Fatalf("erro inesperado: %v", err)
    }
    if order.Amount != 42 {
        t.Errorf("amount = %v, want 42", order.Amount)
    }
}
```

**Integração** (mais lento, roda com menos frequência — normalmente só em CI, não a cada save): usa a implementação real do repositório (`postgres.OrderRepo`), contra um banco de verdade — ou containerizado, subido só pra duração do teste:

```go
func TestOrderRepo_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("pulando teste de integração") // `go test -short` pula este teste
    }
    db := setupTestDB(t) // sobe um Postgres via testcontainers-go, ou aponta pro banco de CI
    repo := postgres.NewOrderRepo(db)

    err := repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    if err != nil {
        t.Fatalf("erro salvando: %v", err)
    }

    got, err := repo.FindByID(context.Background(), "1")
    if err != nil {
        t.Fatalf("erro buscando: %v", err)
    }
    if got.Amount != 42 {
        t.Errorf("amount = %v, want 42", got.Amount)
    }
}
```

`testing.Short()` reporta se o teste foi rodado com a flag `-short` (`go test -short ./...`) — o padrão comum é usar isso pra pular testes lentos no dia a dia do desenvolvedor, rodando-os completos só na esteira de CI. `testcontainers-go` é uma biblioteca que sobe um container Docker (um Postgres real, nesse caso) programaticamente, só pra duração daquele teste, e derruba o container automaticamente ao final — dá pra testar contra um banco de verdade sem precisar de um banco compartilhado e sempre ligado só pra rodar teste.

## Ingênuo vs isolado: por que não testar o service direto contra o Postgres

Uma primeira abordagem, sem se preocupar em isolar dependência, testaria a lógica de `OrderService` sempre contra o banco real:

```go
// versão ingênua: todo teste de OrderService precisa de um Postgres rodando
func TestOrderService_FindOrder_Naive(t *testing.T) {
    db := setupTestDB(t)              // precisa de Docker rodando, subir container, aplicar migration...
    repo := postgres.NewOrderRepo(db)
    repo.Save(context.Background(), domain.Order{ID: "1", Amount: 42})
    service := NewOrderService(repo)

    order, err := service.FindOrder(context.Background(), "1")
    // ... mesma asserção de antes
}
```

Esse teste passa, mas é lento (sobe container a cada execução) e frágil (falha se o Docker não estiver disponível, mesmo quando o bug procurado não tem nada a ver com banco de dados). O motivo de isso ser evitável é a mesma interface `OrderRepository` já vista em [Repository pattern na prática](../repository-pattern-na-pratica/) e em [DDD em Go](../ddd-em-go/): como `OrderService` depende da interface, não da implementação concreta (Dependency Inversion, ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md#d--dependency-inversion-principle)), o teste pode trocar `postgres.OrderRepo` por `FakeOrderRepo` sem precisar mudar uma linha sequer de `OrderService`. O teste unitário passa a validar só a lógica do `OrderService` (o que ele realmente é responsável por testar); a garantia de que `postgres.OrderRepo` de fato conversa certo com o banco fica pro teste de integração, separado.

## Por que a dupla implementação (fake + real) compensa

Isso só é barato de manter porque a interface `OrderRepository` é pequena (poucos métodos) e estável (muda raramente) — trocar de implementação sem editar quem consome é literalmente Open/Closed Principle + Dependency Inversion Principle na prática de teste, não só em produção (ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md)). Se a interface fosse grande e mudasse com frequência, manter duas implementações sincronizadas ficaria caro — outro motivo prático pra preferir interfaces pequenas e focadas (Interface Segregation Principle).

## go test -race

Sempre que o código sob teste envolver concorrência — um worker pool (ver [Padrões de concorrência para infra](../padroes-de-concorrencia-para-infra/)), um cache compartilhado entre goroutines — rode os testes com a flag `-race`:

```
go test -race ./...
```

Isso ativa o **detector de condição de corrida** embutido no Go: uma ferramenta que instrumenta o binário de teste pra detectar, em tempo de execução, quando duas goroutines acessam a mesma variável ao mesmo tempo sem sincronização (sem `sync.Mutex`, sem channel) — um bug que, sem essa ferramenta, pode passar despercebido por um teste sequencial normal (o teste "passa" mesmo com o bug presente, porque a condição de corrida às vezes não se manifesta de forma visível na hora exata da execução).
