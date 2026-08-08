# Clean Architecture / separação em camadas

## O que é Clean Architecture, resumidamente

Clean Architecture é uma forma de organizar um programa em **camadas**, com uma regra única que amarra tudo: **dependência de código sempre aponta pra dentro, do detalhe pra regra de negócio — nunca o contrário.** "Dependência" aqui quer dizer, literalmente, "quem importa o quê" no código. A ideia central está detalhada em [contexts/common/CLEAN-ARCHITECTURE.md](../../contexts/common/CLEAN-ARCHITECTURE.md); o mesmo espírito também aparece com outros nomes (Arquitetura Hexagonal / Ports & Adapters, Onion Architecture) — nomes diferentes pra basicamente a mesma regra.

As camadas, resumidas:

- **Domínio**: a regra de negócio pura — entidades, value objects, regras (ver [DDD em Go](../ddd-em-go/)). Não importa nada de fora: nenhum banco, nenhum framework, nenhuma biblioteca de rede.
- **Aplicação / caso de uso**: orquestra o domínio pra realizar uma operação completa (ex: "criar pedido", "cancelar pedido"), sem ainda saber qual banco ou framework está por trás.
- **Infraestrutura**: a implementação concreta de tudo que o domínio e a aplicação declararam como abstração — banco de dados, fila de mensagens, chamada a uma API externa, cache.
- **Interface / entrega (delivery)**: como o mundo de fora aciona o caso de uso — uma requisição HTTP, um comando de linha, uma chamada gRPC. Essa camada deveria ser trocável (HTTP hoje, CLI amanhã) sem precisar alterar nem o domínio nem a aplicação.

O teste prático mais usado pra saber se essa separação está sendo respeitada: **"dá pra testar a regra de negócio sem precisar de um banco (ou de um servidor) rodando?"** Se a resposta é não, alguma camada está "vazando" pra dentro de outra que deveria ser independente dela.

## Por que isso importa, com um exemplo concreto de violação

Antes de ver o "como fazer certo" em Go, vale ver claramente o que acontece quando essa regra é ignorada — porque a maioria dos projetos não começa "errado" de propósito, começa misturando camadas por conveniência no início e vai piorando aos poucos.

### Versão que mistura camadas (evitar)

```go
package pedido

import (
    "database/sql"
    "encoding/json"
    "net/http"
)

// CancelarPedidoHandler faz TUDO junto: lê a requisição HTTP, valida,
// executa SQL direto, e escreve a resposta HTTP. Regra de negócio,
// infraestrutura e entrega, tudo misturado na mesma função.
func CancelarPedidoHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.URL.Query().Get("id")

        var status string
        err := db.QueryRow("SELECT status FROM pedidos WHERE id = ?", id).Scan(&status)
        if err != nil {
            http.Error(w, "pedido não encontrado", http.StatusNotFound)
            return
        }

        // a regra de negócio "não pode cancelar se já foi enviado"
        // está espremida no meio de código de banco e código HTTP
        if status == "enviado" {
            http.Error(w, "pedido já enviado, não pode cancelar", http.StatusConflict)
            return
        }

        _, err = db.Exec("UPDATE pedidos SET status = 'cancelado' WHERE id = ?", id)
        if err != nil {
            http.Error(w, "erro ao cancelar", http.StatusInternalServerError)
            return
        }

        json.NewEncoder(w).Encode(map[string]string{"status": "cancelado"})
    }
}
```

O problema não é "isso não funciona" — funciona. O problema aparece quando você tenta **testar a regra** "não pode cancelar pedido já enviado" isoladamente: não dá. Pra rodar esse teste, você precisa de um banco de verdade populado E de simular uma requisição HTTP completa, só pra chegar até a linha que realmente importa. E se amanhã essa mesma regra precisar rodar também a partir de um comando de linha (CLI) ou de uma tarefa agendada, ela precisa ser copiada e colada, porque está presa dentro de um `http.HandlerFunc`.

### Versão em camadas (correta)

```go
// pedido/domain.go — camada de domínio. Zero import de banco ou HTTP.
package pedido

import "errors"

type Status string

const (
    StatusAberto    Status = "aberto"
    StatusEnviado   Status = "enviado"
    StatusCancelado Status = "cancelado"
)

type Pedido struct {
    ID     string
    Status Status
}

func (p *Pedido) Cancelar() error {
    if p.Status == StatusEnviado {
        return errors.New("pedido já enviado, não pode cancelar")
    }
    p.Status = StatusCancelado
    return nil
}
```

```go
// pedido/repository.go — a abstração de persistência, pertence ao domínio.
package pedido

type PedidoRepository interface {
    BuscarPorID(id string) (*Pedido, error)
    Salvar(Pedido) error
}
```

```go
// pedido/service.go — camada de aplicação/caso de uso. Orquestra o
// domínio + o repositório, sem saber qual banco existe por trás.
package pedido

type PedidoService struct {
    repo PedidoRepository
}

func NewPedidoService(repo PedidoRepository) *PedidoService {
    return &PedidoService{repo: repo}
}

func (s *PedidoService) Cancelar(id string) error {
    p, err := s.repo.BuscarPorID(id)
    if err != nil {
        return err
    }
    if err := p.Cancelar(); err != nil { // a regra vive na entidade, não aqui
        return err
    }
    return s.repo.Salvar(*p)
}
```

```go
// infra/postgres/pedido_repo.go — só aqui entra database/sql.
package postgres

import (
    "database/sql"
    "meuprojeto/pedido"
)

type PedidoRepo struct{ db *sql.DB }

func (r *PedidoRepo) BuscarPorID(id string) (*pedido.Pedido, error) {
    var status string
    err := r.db.QueryRow("SELECT status FROM pedidos WHERE id = ?", id).Scan(&status)
    if err != nil {
        return nil, err
    }
    return &pedido.Pedido{ID: id, Status: pedido.Status(status)}, nil
}

func (r *PedidoRepo) Salvar(p pedido.Pedido) error {
    _, err := r.db.Exec("UPDATE pedidos SET status = ? WHERE id = ?", p.Status, p.ID)
    return err
}
```

```go
// infra/http/pedido_handler.go — só aqui entra net/http.
package http

import (
    "encoding/json"
    "net/http"
    "meuprojeto/pedido"
)

func CancelarPedidoHandler(service *pedido.PedidoService) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.URL.Query().Get("id")
        if err := service.Cancelar(id); err != nil {
            http.Error(w, err.Error(), http.StatusConflict)
            return
        }
        json.NewEncoder(w).Encode(map[string]string{"status": "cancelado"})
    }
}
```

```go
// main.go — o único lugar que conhece TODAS as camadas ao mesmo tempo,
// e monta a aplicação inteira conectando as peças.
func main() {
    db, _ := sql.Open("postgres", "...")
    repo := &postgres.PedidoRepo{db: db}
    service := pedido.NewPedidoService(repo)
    handler := http.CancelarPedidoHandler(service)
    // registra handler no servidor, sobe o servidor, etc.
}
```

Agora `p.Cancelar()` (a regra de negócio de verdade) é testável sozinha, sem banco e sem servidor HTTP no ar — é só criar um `Pedido{Status: StatusEnviado}` na mão e chamar `.Cancelar()`, checando que devolve erro. E se amanhã aparecer um comando de linha que também precisa cancelar pedidos, ele reusa exatamente o mesmo `PedidoService`, sem duplicar a regra.

## O que faz essa regra "pegar" de verdade em Go

Aqui está o ponto mais importante de como isso funciona especificamente em Go, e é o mesmo mecanismo que sustenta boa parte de SOLID (ver Dependency Inversion em [SOLID em Go](../solid-em-go/)): a interface `PedidoRepository` é declarada dentro do pacote `pedido` (o domínio), e o pacote `postgres` (a infraestrutura) é quem satisfaz essa interface — de forma implícita, sem nenhuma linha de código dizendo "postgres implementa pedido.PedidoRepository". Ninguém no domínio importa `database/sql`. Ninguém no domínio sabe que Postgres existe.

Vale ser honesto sobre uma coisa: nada no compilador de Go **impede fisicamente** alguém de escrever `import "database/sql"` dentro do arquivo `pedido/domain.go` por engano ou por pressa — o compilador não tem noção nenhuma de "camada" ou "arquitetura", ele só sabe compilar o que você escreveu. O que torna essa regra difícil de violar por acidente é que o **caminho natural de menor esforço em Go já é o caminho certo**: como a interface fica perto de quem consome (o domínio) e é satisfeita implicitamente, escrever o código "do jeito certo" não dá mais trabalho do que escrever errado — ao contrário, geralmente dá menos, porque você não precisa lidar com um `*sql.DB` real dentro de testes de regra de negócio. Isso é diferente de linguagens onde nada no design da linguagem empurra você pra esse caminho, e a separação em camadas depende inteiramente de disciplina de equipe e revisão de código pra não ser violada silenciosamente.

## O teste rápido, aplicado de verdade

A pergunta "dá pra testar o domínio sem banco/servidor no ar?" vira, em Go, uma checagem quase mecânica: o arquivo `domain.go` (ou qualquer arquivo de domínio) não deveria ter, em lugar nenhum, um `import "database/sql"` nem um `import "net/http"`. Isso é rápido de verificar sem nem precisar rodar teste nenhum:

```
grep -r "database/sql\|net/http" pedido/domain.go pedido/repository.go
```

Se esse comando não encontra nada, a regra de dependência está sendo respeitada nesses arquivos. É um teste tosco (não substitui revisão de código de verdade, e não pega toda violação possível — por exemplo, não pega uma dependência indireta escondida atrás de outro pacote), mas é rápido o suficiente pra rodar sempre que quiser, e serve como um primeiro sinal de alerta bem barato.

## Por que vale desenhar isso desde cedo em Go

Em ambientes onde é barato "remendar" código depois — trocar o tipo de uma dependência, adicionar uma abstração que não existia, sem precisar reescrever assinatura de função em vários lugares — misturar camadas no começo de um projeto pequeno e desembaraçar depois não é tão caro. Go, sendo uma linguagem compilada e com tipagem estática (ver [O que é Go](../o-que-e-go/)), não tem esse tipo de remendo "de emergência": mudar uma assinatura de função te obriga a atualizar cada lugar que a chama, e o compilador te avisa exatamente onde. Isso não é ruim — é inclusive uma proteção — mas significa que, num serviço que já nasce sabendo que vai crescer (que é justamente o caso de uma API real com banco de dados por trás), vale desenhar a fronteira entre camadas desde o início, em vez de esperar o projeto já ter crescido o suficiente pra doer.

Isso **não** significa que todo exercício pequeno de 20 linhas precisa de 4 pastas (`domain/`, `application/`, `infra/`, `delivery/`) com uma interface pra cada dependência. Um script pequeno, sem plano de crescer, ganha muito pouco com esse aparato todo — a mesma lógica de "não force um padrão pesado numa peça pequena que não precisa dele" que já apareceu em [SOLID em Go](../solid-em-go/) e em [DDD em Go](../ddd-em-go/).

## Onde isso aparece no resto do plano

[Repository pattern na prática](../repository-pattern-na-pratica/) (Módulo 3) é exatamente esta mesma ideia — interface de persistência no domínio, implementação concreta na infra — só que com um exemplo mais completo, incluindo uma implementação Postgres real e uma implementação falsa (em memória) usada só para testes. [net/http e APIs REST](../net-http-e-apis-rest/) e [Middleware e chain de handlers](../middleware-e-chain-de-handlers/) são a camada de entrega: HTTP é só **um** jeito de acionar o caso de uso — poderia ser um comando de linha (CLI) ou gRPC sem precisar mudar uma linha sequer do domínio.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise clean architecture`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
