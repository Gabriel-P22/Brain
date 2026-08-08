# DDD em Go

## O que é DDD, resumidamente

DDD (Domain-Driven Design, ou "design orientado ao domínio") é uma forma de organizar código que parte de uma ideia simples: o código deveria ser modelado em cima da linguagem real do negócio (o "domínio" — as regras, os termos, os processos que existem independente de qualquer computador), não em cima da estrutura de tabela do banco de dados nem do formato do endpoint HTTP. Se um analista de negócio fala em "pedido", "item", "cancelamento", o código deveria ter um tipo `Pedido`, um conceito de item, e uma operação de cancelamento — não só colunas numa tabela `pedidos`.

A definição completa de cada bloco tático (as peças que DDD usa pra montar esse modelo) está em [contexts/common/DDD.md](../../contexts/common/DDD.md) — aqui vai um resumo rápido de cada peça, e depois o que muda especificamente ao escrever isso em Go.

- **Entidade**: um tipo que é definido pela sua identidade (um ID), não pelo conteúdo dos seus campos. Dois objetos com o mesmo ID são "a mesma coisa", mesmo que os outros campos tenham mudado ao longo do tempo. A regra de negócio de uma entidade deveria morar dentro dos métodos dela mesma, não espalhada em outro lugar.
- **Value Object (Objeto de Valor)**: o oposto — não tem identidade própria, é definido inteiramente pelo valor dos seus campos. Dois Value Objects com os mesmos valores são intercambiáveis (por exemplo, dois valores monetários de "R$ 10,00" são exatamente a mesma coisa). Costuma ser imutável: uma operação sobre ele devolve um Value Object novo, em vez de alterar o que já existia.
- **Agregado**: um grupo de entidades e value objects que precisa mudar sempre junto, como uma unidade só, com uma **raiz** (a "aggregate root") sendo o único ponto de entrada — nada de fora do agregado deveria alterar uma peça interna dele diretamente, só passando pela raiz.
- **Repository**: uma abstração de persistência — uma interface que expõe operações do vocabulário do domínio (tipo `Salvar`, `BuscarPorID`), sem vazar detalhe de como o dado é guardado de verdade (SQL? arquivo? outra API?).
- **Domain Service**: uma regra de negócio que envolve mais de uma entidade ao mesmo tempo, e por isso não pertence naturalmente a nenhuma delas sozinha.
- **Bounded Context**: uma fronteira estratégica — dentro dela, um termo do negócio (ex: "Pedido") tem um único significado consistente; fora dela, o "mesmo" termo pode significar outra coisa.

Vamos entender cada um com exemplo de código Go, começando pelo mais importante de entender bem: a diferença entre um modelo de domínio "rico" (o que DDD quer) e um "anêmico" (o erro mais comum).

## O erro mais comum: modelo de domínio anêmico

Um "modelo anêmico" é quando o tipo que representa o conceito de negócio (a entidade) vira só um saco de campos — sem nenhuma regra própria — e toda a lógica de negócio fica espalhada em funções ou tipos "de serviço" separados, que ficam manipulando os campos da entidade de fora pra dentro.

### Versão anêmica (evitar)

```go
package pedido

// Pedido é só um saco de dados — nenhuma regra mora aqui dentro.
type Pedido struct {
    ID     string
    Status string
    Itens  []string
}

// PedidoService concentra TODA a regra de negócio, mexendo nos campos
// do Pedido de fora, como se Pedido fosse uma struct burra.
type PedidoService struct{}

func (s PedidoService) Cancelar(p *Pedido) error {
    if p.Status == "enviado" {
        return errors.New("pedido já enviado, não pode cancelar")
    }
    p.Status = "cancelado"
    return nil
}

func (s PedidoService) AdicionarItem(p *Pedido, item string) error {
    if len(p.Itens) >= 50 {
        return errors.New("limite de itens excedido")
    }
    p.Itens = append(p.Itens, item)
    return nil
}
```

Esse código funciona, mas tem um problema estrutural: nada impede outro pedaço do sistema de escrever `p.Status = "cancelado"` diretamente, sem passar pelo `PedidoService.Cancelar` — e aí a checagem "não pode cancelar se já foi enviado" é simplesmente ignorada, silenciosamente, porque a regra nunca esteve realmente "presa" ao tipo `Pedido`. Quanto mais o sistema cresce, mais lugares diferentes acabam manipulando `Pedido.Status` direto, cada um reimplementando (ou esquecendo de implementar) a mesma regra.

### Versão com modelo rico (correta em DDD)

```go
package pedido

import "errors"

type Status string

const (
    StatusAberto     Status = "aberto"
    StatusEnviado    Status = "enviado"
    StatusCancelado  Status = "cancelado"
)

// Pedido agora carrega a regra de negócio dentro de si mesmo.
type Pedido struct {
    ID     string
    Status Status
    Itens  []string
}

// Cancelar é um método do PRÓPRIO Pedido — a regra "não cancela se já
// foi enviado" está fisicamente presa ao tipo, não pode ser esquecida
// em outro lugar do sistema.
func (p *Pedido) Cancelar() error {
    if p.Status == StatusEnviado {
        return errors.New("pedido já enviado, não pode cancelar")
    }
    p.Status = StatusCancelado
    return nil
}

func (p *Pedido) AdicionarItem(item string) error {
    if len(p.Itens) >= 50 {
        return errors.New("limite de itens excedido")
    }
    p.Itens = append(p.Itens, item)
    return nil
}
```

Agora, chamar `pedido.Cancelar()` é a única forma normal de cancelar — a regra está garantida onde o dado vive. Isso não elimina a necessidade de um "service" de aplicação (você ainda pode ter um `PedidoService` que orquestra: busca o pedido no banco, chama `pedido.Cancelar()`, salva de volta) — a diferença é que esse service **orquestra**, ele não **reimplementa** a regra de negócio por fora.

## Entidade — identidade, não conteúdo

Uma entidade é "a mesma" ao longo do tempo mesmo que seus campos mudem, porque o que a identifica é o `ID`, não o conjunto de valores atual.

```go
package pedido

type Pedido struct {
    ID     string // é isso que define "quem" é este pedido
    Status Status
}

func (p *Pedido) Cancelar() error {
    // ... regra, como visto acima
    return nil
}
```

Dois valores `Pedido{ID: "abc", Status: StatusAberto}` e `Pedido{ID: "abc", Status: StatusCancelado}` são **o mesmo pedido** em dois momentos diferentes — mesmo `ID`, campos diferentes. Já dois pedidos com IDs diferentes são pedidos diferentes, mesmo que todos os outros campos sejam idênticos. Isso é o oposto de um Value Object, que você vê a seguir.

## Value Object — só o valor importa

Um Value Object não tem ID nenhum. Dois Value Objects "iguais" (mesmo conteúdo) são completamente intercambiáveis — não faz sentido perguntar "mas são o mesmo objeto?", porque a pergunta não tem significado pra esse tipo de dado.

```go
package dinheiro

import "errors"

// Money não tem ID. Dois Money{Amount: 1000, Currency: "BRL"} são
// exatamente a mesma coisa — não existe "identidade" pra dinheiro.
type Money struct {
    Amount   int64  // guardado em centavos, pra evitar problema de arredondamento com número decimal
    Currency string
}

// Add não muda o Money original — devolve um novo. Isso é imutabilidade,
// uma convenção forte em Value Object: operação nunca muta o que já existe.
func (m Money) Add(outro Money) (Money, error) {
    if m.Currency != outro.Currency {
        return Money{}, errors.New("moedas diferentes, não dá pra somar")
    }
    return Money{Amount: m.Amount + outro.Amount, Currency: m.Currency}, nil
}
```

Repare na assinatura de `Add`: o receiver é `m Money` (valor, não ponteiro) — porque `Money` é pequeno e imutável, copiar é barato e desejável (evita alguém mutar por engano um `Money` que estava sendo compartilhado em outro lugar do programa). Isso é uma aplicação direta da regra "receiver por valor quando é só leitura e o struct é pequeno" que você já viu em [Structs, métodos e interfaces](../structs-metodos-e-interfaces/).

## Agregado — uma raiz, um ponto de entrada

Um agregado é um grupo de entidades/value objects que deve mudar sempre em conjunto, como uma transação lógica só. A raiz do agregado é o único ponto de entrada permitido pra mexer no que está dentro dele.

```go
package pedido

import "errors"

// ItemPedido só existe dentro de um Pedido — ele não tem repositório
// próprio, não é buscado nem salvo separadamente do Pedido.
type ItemPedido struct {
    Produto    string
    Quantidade int
}

// Pedido é a RAIZ do agregado — o único ponto de entrada pra mexer
// nos itens.
type Pedido struct {
    ID    string
    Itens []ItemPedido
}

// AdicionarItem é o único jeito "certo" de colocar um item na lista.
// Ninguém deveria fazer pedido.Itens = append(pedido.Itens, ...) direto
// de fora deste pacote, porque aí a regra do limite seria ignorada.
func (p *Pedido) AdicionarItem(item ItemPedido) error {
    if len(p.Itens) >= 50 {
        return errors.New("limite de itens excedido")
    }
    p.Itens = append(p.Itens, item)
    return nil
}
```

Aqui chega o ponto mais importante (e mais delicado) de como isso funciona especificamente em Go: **Go não tem um jeito de forçar, pelo compilador, que ninguém mexa em `p.Itens` direto de fora do pacote.** Em outras linguagens, você declararia o campo `Itens` como "privado" (visível só dentro da própria classe) e o compilador bloquearia qualquer tentativa de acesso direto de fora. Go não tem essa granularidade — a visibilidade em Go é por **pacote inteiro**, não por tipo: um campo com letra minúscula (`itens`, não `Itens`) fica invisível pra qualquer código *fora do pacote* `pedido`, mas continua totalmente acessível pra qualquer código *dentro* do pacote `pedido`, mesmo que esse código esteja em outro arquivo, num tipo diferente. Ou seja: dentro do próprio pacote não existe "privado de verdade" entre tipos — é convenção de design, reforçada pela organização de pacotes, não uma trava do compilador.

Na prática, isso quer dizer: se você realmente quer impedir que código externo escreva direto em `Itens`, o campo precisa começar com letra minúscula (`itens`) e você expõe só métodos como `AdicionarItem` e talvez `Itens() []ItemPedido` (um getter que devolve uma cópia, não o slice original) pra leitura controlada. A fronteira do agregado em Go é, então, uma combinação de: (1) visibilidade de pacote (letra minúscula) + (2) convenção de design (documentar e revisar em code review que ninguém deveria burlar isso, mesmo quando tecnicamente poderia, estando no mesmo pacote).

## Repository — a abstração de persistência

```go
package pedido

// PedidoRepository é declarado AQUI, no pacote de domínio — não no
// pacote que vai implementar o banco de verdade. Isso é Dependency
// Inversion (ver contexts/common/SOLID.md#d--dependency-inversion-principle
// e SOLID em Go) aplicado especificamente ao conceito de Repository do DDD.
type PedidoRepository interface {
    Salvar(Pedido) error
    BuscarPorID(id string) (*Pedido, error)
}
```

Um detalhe que costuma surpreender quem vem de linguagens com um ORM (mapeador objeto-relacional) muito ativo, que já cuida de converter automaticamente entre o objeto de domínio e a linha da tabela: em Go, não existe isso de fábrica na biblioteca padrão. `database/sql` (que você vai estudar no Módulo 3, em [Acesso a dados e context.Context](../acesso-a-dados-e-context/)) trabalha em cima de linhas e colunas cruas — normalmente você mesmo escreve uma função que converte a linha do banco pra um `Pedido` de domínio, e vice-versa, dentro da implementação concreta do `PedidoRepository`. Isso é mais código escrito à mão do que em ambientes com ORM ativo, mas é exatamente o que mantém o tipo `Pedido` (o domínio) livre de qualquer `import` de banco de dados — o domínio não sabe, nem deveria saber, como ele é guardado.

```go
package postgres

import "database/sql"

// PedidoRow representa a LINHA da tabela — um tipo separado do Pedido
// de domínio, de propósito.
type pedidoRow struct {
    ID     string
    Status string
}

type PedidoRepo struct{ db *sql.DB }

func (r *PedidoRepo) BuscarPorID(id string) (*pedido.Pedido, error) {
    var row pedidoRow
    err := r.db.QueryRow("SELECT id, status FROM pedidos WHERE id = ?", id).
        Scan(&row.ID, &row.Status)
    if err != nil {
        return nil, err
    }
    // conversão explícita: linha do banco -> entidade de domínio
    return &pedido.Pedido{ID: row.ID, Status: pedido.Status(row.Status)}, nil
}
```

## Domain Service — regra que não pertence a uma entidade só

Às vezes uma regra de negócio envolve duas entidades ao mesmo tempo, e forçá-la a morar dentro de uma delas seria artificial.

```go
package conta

import "errors"

type Conta struct {
    ID    string
    Saldo Money
}

func (c *Conta) Sacar(valor Money) error {
    if c.Saldo.Amount < valor.Amount {
        return errors.New("saldo insuficiente")
    }
    c.Saldo.Amount -= valor.Amount
    return nil
}

func (c *Conta) Depositar(valor Money) {
    c.Saldo.Amount += valor.Amount
}

// TransferirEntreContas não pertence naturalmente nem a "conta origem"
// nem a "conta destino" sozinha — ela orquestra as duas. Isso é um
// Domain Service: ainda é regra de negócio pura (não sabe nada de banco,
// não importa net/http nem database/sql), só que não é método de
// nenhuma entidade específica.
func TransferirEntreContas(origem, destino *Conta, valor Money) error {
    if err := origem.Sacar(valor); err != nil {
        return err
    }
    destino.Depositar(valor)
    return nil
}
```

## Bounded Context — fronteira de significado

Um Bounded Context é uma fronteira estratégica: dentro dela, um termo do negócio tem um significado único; fora dela, o "mesmo" termo pode significar algo completamente diferente. Um exemplo clássico: o "Pedido" que interessa ao time de **cobrança** (valor total, status de pagamento, forma de pagamento) não é a mesma coisa que o "Pedido" que interessa ao time de **logística** (endereço de entrega, status de separação no estoque, transportadora) — mesmo que os dois times usem exatamente a palavra "Pedido" no dia a dia.

Em Go, um Bounded Context tende a virar um **pacote de alto nível**, separado fisicamente dos outros:

```
cobranca/
  pedido.go   // type Pedido struct{ ID string; Valor Money; StatusPagamento string }

logistica/
  pedido.go   // type Pedido struct{ ID string; Endereco string; StatusEntrega string }
```

Repare que os dois pacotes têm um tipo chamado `Pedido`, e isso não é um problema — é exatamente o ponto. Cada pacote tem seu próprio `Pedido`, moldado só com os campos que fazem sentido pro contexto dele. O erro estratégico mais comum é o oposto disso: criar **um único tipo `Pedido` compartilhado** entre `cobranca` e `logistica`, indo colecionando campos de ambos os times ao longo do tempo — isso mistura dois modelos mentais diferentes numa estrutura só, e qualquer mudança pedida por um time passa a arriscar quebrar o outro.

## Quando NÃO aplicar DDD tático completo

Nem todo pacote pequeno precisa de Entidade + Value Object + Agregado + Repository + Domain Service, todos formalizados. Para um CRUD simples sem regra de negócio real por trás (por exemplo, uma tabela de "categorias" que só tem nome e é só listada/criada/editada, sem nenhuma regra além de "nome não pode ser vazio"), forçar todo esse aparato é over-engineering — a mesma lógica de "não force um padrão pesado numa peça pequena que não precisa dele" que já apareceu em [SOLID em Go](../solid-em-go/). DDD tático compensa quando existe complexidade de regra de negócio genuína pra proteger (várias regras de validação, transições de estado, cálculos que dependem de múltiplos campos) — não pela complexidade do padrão em si.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise ddd em go`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
