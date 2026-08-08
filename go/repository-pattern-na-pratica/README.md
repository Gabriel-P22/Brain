# Repository pattern na prática

## O que é o Repository pattern

Um **repositório** (repository) é uma abstração de persistência: uma interface que expõe operações no vocabulário do domínio (`Save`, `FindByID`, "salvar um pedido", "buscar um pedido pelo id") sem vazar nenhum detalhe de **como** o dado é de fato guardado — se é um banco relacional, um arquivo, um serviço externo, ou até só uma estrutura em memória. A definição completa está em [contexts/common/DDD.md](../../contexts/common/DDD.md#repository) — este tópico é a aplicação prática dela em Go, com um exemplo completo.

O ponto central do padrão: **a mesma interface pode ter várias implementações diferentes**, e quem usa o repositório (a camada de aplicação, também chamada de camada de casos de uso) não sabe nem se importa com qual implementação concreta está por trás — ela só conhece a interface. Isso é [Dependency Inversion Principle](../../contexts/common/SOLID.md#d--dependency-inversion-principle) aplicado de forma bem concreta: a abstração (`OrderRepository`) pertence ao domínio, que é quem consome; a implementação concreta (banco de dados de verdade) mora na infraestrutura, e depende do domínio — nunca o contrário. [SOLID em Go](../solid-em-go/) e [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/) já cobriram essa ideia em teoria; aqui é o exemplo de ponta a ponta.

## Por que isso funciona em Go sem herança nem framework

Em Go, uma struct **satisfaz uma interface automaticamente**, só por ter os métodos com a assinatura certa — não existe uma palavra-chave tipo `implements` que você precisa escrever explicitamente (isso já foi visto em [Structs, métodos e interfaces](../structs-metodos-e-interfaces/)). É essa característica da linguagem — chamada de **tipagem estrutural implícita** — que permite ter duas implementações completamente diferentes (uma com banco de dados real, uma só em memória para teste) da mesma interface, sem precisar de nenhum container de injeção de dependência nem de nenhuma anotação especial. A "injeção" é, literalmente, passar o valor certo como argumento de uma função construtora.

## A interface, no domínio

```go
// domain/order.go
package domain

import (
    "context"
    "errors"
)

// Order é a entidade de domínio — ver DDD em Go para o que caracteriza uma entidade.
type Order struct {
    ID     string
    Amount float64
}

// OrderRepository é a abstração de persistência. Ela é declarada no pacote de domínio,
// não no pacote de infraestrutura — quem "dona" da interface é quem consome, não quem
// implementa (Dependency Inversion).
type OrderRepository interface {
    Save(ctx context.Context, o Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}

// ErrNotFound é um erro sentinela — uma variável de erro exportada que qualquer
// implementação de OrderRepository deve devolver quando o pedido não existe, para que
// quem chama consiga checar "não achei" de forma consistente, sem depender de detalhe
// de qual implementação está sendo usada. Ver Tratamento de erros e pacotes.
var ErrNotFound = errors.New("pedido não encontrado")
```

## A camada de aplicação: o caso de uso

Entre o domínio e a infraestrutura, existe a camada de **aplicação** (ou "caso de uso") — ela orquestra o domínio para realizar uma operação completa, mas ainda não sabe qual banco de dados está por trás:

```go
// app/order_service.go
package app

import (
    "context"
    "fmt"

    "meuprojeto/domain"
)

// OrderService depende só da interface — não importa nada de infra/postgres.
type OrderService struct {
    repo domain.OrderRepository
}

// NewOrderService é o "ponto de injeção": quem chama decide qual implementação passar.
func NewOrderService(repo domain.OrderRepository) *OrderService {
    return &OrderService{repo: repo}
}

func (s *OrderService) CriarPedido(ctx context.Context, id string, valor float64) error {
    pedido := domain.Order{ID: id, Amount: valor}
    if err := s.repo.Save(ctx, pedido); err != nil {
        return fmt.Errorf("salvando pedido: %w", err)
    }
    return nil
}

func (s *OrderService) BuscarPedido(ctx context.Context, id string) (*domain.Order, error) {
    pedido, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("buscando pedido %s: %w", id, err)
    }
    return pedido, nil
}
```

## Implementação real, na infraestrutura

```go
// infra/postgres/order_repo.go
package postgres

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "meuprojeto/domain"
)

type OrderRepo struct {
    db *sql.DB
}

func NewOrderRepo(db *sql.DB) *OrderRepo {
    return &OrderRepo{db: db}
}

func (r *OrderRepo) Save(ctx context.Context, o domain.Order) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO orders (id, amount) VALUES ($1, $2)
         ON CONFLICT (id) DO UPDATE SET amount = $2`,
        o.ID, o.Amount)
    return err
}

func (r *OrderRepo) FindByID(ctx context.Context, id string) (*domain.Order, error) {
    var o domain.Order
    err := r.db.QueryRowContext(ctx, "SELECT id, amount FROM orders WHERE id = $1", id).
        Scan(&o.ID, &o.Amount)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, fmt.Errorf("pedido %s: %w", id, domain.ErrNotFound)
    }
    if err != nil {
        return nil, err
    }
    return &o, nil
}
```

Repare que `*postgres.OrderRepo` nunca declara "eu implemento `domain.OrderRepository`" em lugar nenhum do código — o compilador Go confirma isso sozinho, no momento em que alguém tenta usar um `*postgres.OrderRepo` onde um `domain.OrderRepository` é esperado (por exemplo, passando ele para `NewOrderService`). Se algum método estivesse faltando ou com assinatura diferente, o erro apareceria ali, na hora de montar essa peça, não dentro do pacote `postgres`.

`sql.ErrNoRows` é o erro que o `database/sql` devolve quando uma query com `QueryRow` não encontra nenhuma linha (ver [Acesso a dados e context.Context](../acesso-a-dados-e-context/)) — a implementação Postgres traduz esse detalhe técnico específico de banco para o erro de domínio `domain.ErrNotFound`, envolvido com `%w` para preservar a cadeia original (ver `errors.Is`/`errors.As` em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)). Essa tradução é importante: quem consome `OrderService` não deveria precisar saber que "por trás" existe um `sql.ErrNoRows` — só precisa saber checar `domain.ErrNotFound`, que é um conceito de domínio, não de banco.

## Implementação fake, para teste

```go
// domain/order_repo_fake.go
package domain

import (
    "context"
    "sync"
)

// FakeOrderRepo implementa a mesma interface OrderRepository, guardando tudo num map
// em memória — sem banco de dados real envolvido.
type FakeOrderRepo struct {
    mu   sync.Mutex
    data map[string]Order
}

func NewFakeOrderRepo() *FakeOrderRepo {
    return &FakeOrderRepo{data: make(map[string]Order)}
}

func (r *FakeOrderRepo) Save(ctx context.Context, o Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.data[o.ID] = o
    return nil
}

func (r *FakeOrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    o, ok := r.data[id]
    if !ok {
        return nil, ErrNotFound
    }
    return &o, nil
}
```

## Usando o fake num teste, sem banco no ar

```go
// app/order_service_test.go
package app_test

import (
    "context"
    "errors"
    "testing"

    "meuprojeto/app"
    "meuprojeto/domain"
)

func TestOrderService_CriarEBuscarPedido(t *testing.T) {
    repo := domain.NewFakeOrderRepo()
    service := app.NewOrderService(repo)
    ctx := context.Background()

    if err := service.CriarPedido(ctx, "42", 350.0); err != nil {
        t.Fatalf("erro inesperado criando pedido: %v", err)
    }

    pedido, err := service.BuscarPedido(ctx, "42")
    if err != nil {
        t.Fatalf("erro inesperado buscando pedido: %v", err)
    }
    if pedido.Amount != 350.0 {
        t.Errorf("valor = %v, esperado 350.0", pedido.Amount)
    }
}

func TestOrderService_BuscarPedidoInexistente(t *testing.T) {
    repo := domain.NewFakeOrderRepo()
    service := app.NewOrderService(repo)

    _, err := service.BuscarPedido(context.Background(), "não-existe")
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("esperava domain.ErrNotFound, recebeu: %v", err)
    }
}
```

Esse teste roda em milissegundos, sem subir nenhum banco de dados, sem mock framework nenhum — `FakeOrderRepo` é só um struct comum que satisfaz a mesma interface que `*postgres.OrderRepo` satisfaz. É o "dá para testar isso isolado?" do [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/), concretizado: a regra de negócio de `OrderService` é testável sem depender de infraestrutura nenhuma.

## Montando tudo em main.go

A única peça do programa que conhece **as duas pontas** — a interface e a implementação concreta — é o ponto de entrada da aplicação, `main.go`. É aqui que a "injeção de dependência" realmente acontece, sem nenhum framework:

```go
// main.go
package main

import (
    "database/sql"
    "log"

    _ "github.com/lib/pq"

    "meuprojeto/app"
    "meuprojeto/infra/postgres"
)

func main() {
    db, err := sql.Open("postgres", "postgres://user:pass@localhost/meubanco")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := postgres.NewOrderRepo(db)     // implementação concreta, escolhida aqui
    service := app.NewOrderService(repo)  // OrderService só enxerga domain.OrderRepository

    // service agora está pronto para ser usado por um handler HTTP, um worker, etc.
    _ = service
}
```

Se amanhã o time decidir trocar Postgres por outro banco, ou por uma API de armazenamento gerenciado, a mudança fica isolada: uma nova struct em `infra/`, satisfazendo a mesma interface `domain.OrderRepository`, e uma linha trocada aqui em `main.go`. `domain/` e `app/` não precisam mudar uma linha sequer.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise repository pattern na prática`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: adicionar um método `Delete(ctx, id) error` na interface `OrderRepository`, implementá-lo nas duas implementações (Postgres e fake), e escrever um teste que confirma que buscar um pedido depois de deletá-lo devolve `domain.ErrNotFound`.
