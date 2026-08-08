# SOLID em Go

## O que é SOLID

SOLID é um conjunto de 5 princípios de design de software, cada um representado por uma letra da sigla. A ideia central de todos eles é a mesma: escrever código que seja fácil de entender, fácil de mudar sem quebrar coisas que já funcionavam, e fácil de testar. Cada princípio ataca esse problema de um ângulo diferente.

A definição formal e agnóstica de cada um está em [contexts/common/SOLID.md](../../contexts/common/SOLID.md) — este README não repete aquilo palavra por palavra, mas resume o suficiente de cada princípio pra você acompanhar sem precisar abrir outro arquivo no meio da leitura. O exemplo de código "de referência" (mais curto, sem toda a explicação) fica em [go/config/reference/SOLID.md](../config/reference/SOLID.md).

## Por que SOLID em Go é diferente do que você talvez já tenha ouvido falar

SOLID foi descrito originalmente pensando em linguagens que têm **classes** e **herança** — onde um tipo pode "herdar" campos e métodos de outro tipo, formando uma hierarquia (`Cachorro` herda de `Animal`, que herda de `SerVivo`, etc.).

Go **não tem herança de classe**. Go não tem nem "classe", no sentido tradicional da palavra. O que Go tem, no lugar disso, é:

- **`struct`**: um tipo que agrupa dados (campos), sem herança nenhuma envolvida.
- **Métodos ligados a um tipo**: funções associadas a um `struct` (você já viu isso no tópico [Structs, métodos e interfaces](../structs-metodos-e-interfaces/)).
- **Composição**: um `struct` pode conter outro `struct` dentro dele (chamado de *embedding*), mas isso é só um campo — não é herança, mesmo que os métodos do struct interno fiquem acessíveis automaticamente. É importante deixar isso bem claro desde já, porque é um erro comum: usar embedding tentando simular herança, quando o motivo real é só reaproveitar código de outro tipo. Se você pensar "eu quero herdar de X", pare e pergunte: "isso é realmente uma relação 'é um' (`Dog` é um `Animal`), ou eu só quero reusar um pedaço de comportamento?" No segundo caso, o jeito idiomático em Go costuma ser um campo nomeado comum (não embutido) + chamar o método explicitamente, não embedding disfarçado de herança.
- **Interface satisfeita de forma implícita**: você declara um conjunto de métodos que um tipo precisa ter (a interface), e qualquer `struct` que tiver esses métodos satisfaz essa interface automaticamente — sem nenhuma linha de código dizendo "este struct implementa esta interface". Isso é bem diferente de linguagens onde você precisa escrever explicitamente essa relação. Esse mecanismo (interface pequena, implícita, declarada perto de quem consome o dado) é o que faz Go conseguir aplicar boa parte de SOLID sem herança nenhuma — principalmente as letras **S**, **I** e **D**, que são as que você vai ver com mais frequência no dia a dia de Go. **O** e **L** continuam válidos e importantes, mas aparecem com menos frequência explícita, porque o jeito natural de escrever Go (funções pequenas + interfaces pequenas) já evita boa parte das violações clássicas.

Com isso dito, vamos por cada letra.

## S — Single Responsibility Principle (Princípio da Responsabilidade Única)

**Em palavras simples**: cada tipo (cada `struct`) deveria ter um único motivo para mudar. Se você olha pro código e pensa "esse tipo faz três coisas diferentes: calcula, salva no banco e manda e-mail", isso são três responsabilidades que deveriam estar em três lugares — porque um dia você vai precisar mudar só a parte do e-mail, e não quer arriscar quebrar o cálculo ao mexer nesse arquivo.

### Versão ingênua

```go
package relatorio

import (
    "fmt"
    "os"
)

// RelatorioMensal calcula, formata E grava um relatório em disco.
// Três responsabilidades diferentes no mesmo tipo.
type RelatorioMensal struct {
    Vendas []float64
}

func (r RelatorioMensal) Gerar(caminho string) error {
    // responsabilidade 1: calcular
    total := 0.0
    for _, v := range r.Vendas {
        total += v
    }

    // responsabilidade 2: formatar texto
    texto := fmt.Sprintf("Total vendido no mês: R$ %.2f\n", total)

    // responsabilidade 3: gravar em arquivo
    return os.WriteFile(caminho, []byte(texto), 0644)
}
```

O problema não é que esse código está "errado" — ele funciona. O problema aparece depois: se amanhã você precisar trocar "gravar em arquivo" por "enviar por e-mail", ou mudar o formato do texto de "R$ 123.45" pra um formato com separador de milhar, você mexe no mesmo método que também faz a conta. Qualquer engano ali arrisca quebrar o cálculo, que não tinha nada a ver com a mudança pedida. E fica difícil testar só o cálculo sem também precisar de acesso a disco.

### Versão refatorada

```go
package relatorio

import "fmt"

// Calculadora sabe só calcular o total. Nada de formatação, nada de I/O.
type Calculadora struct {
    Vendas []float64
}

func (c Calculadora) Total() float64 {
    total := 0.0
    for _, v := range c.Vendas {
        total += v
    }
    return total
}

// Formatador sabe só transformar um número em texto legível.
type Formatador struct{}

func (Formatador) FormatarTotal(total float64) string {
    return fmt.Sprintf("Total vendido no mês: R$ %.2f\n", total)
}

// Gravador sabe só escrever um texto em algum destino — a interface não
// diz SE é arquivo, e-mail ou banco. Isso já antecipa o D (Dependency
// Inversion), que você vai ver mais abaixo.
type Gravador interface {
    Gravar(texto string) error
}
```

Agora cada tipo tem um único motivo pra mudar: `Calculadora` muda só se a fórmula do total mudar; `Formatador` muda só se o texto de saída mudar; quem grava pode ser trocado (arquivo, e-mail, banco) sem tocar em nenhum dos outros dois. E testar `Calculadora.Total()` sozinho, sem precisar de disco nem rede, fica trivial — o que é exatamente o tipo de testabilidade que Clean Architecture também persegue (ver [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/)).

## O — Open/Closed Principle (Aberto para extensão, fechado para modificação)

**Em palavras simples**: você deveria conseguir adicionar um comportamento novo ao sistema **sem editar** o código que já existe e já funciona — só adicionando código novo ao lado. O sinal clássico de violação é um bloco `switch` (ou uma sequência de `if`/`else if`) que cresce toda vez que aparece um caso novo.

### Versão ingênua

```go
package desconto

// CalcularDesconto cresce a cada tipo novo de cliente que aparece.
// Toda vez que o negócio inventa um tipo de cliente novo, alguém
// precisa vir editar esta função (que já funcionava) pra adicionar um case.
func CalcularDesconto(tipoCliente string, valor float64) float64 {
    switch tipoCliente {
    case "comum":
        return valor
    case "vip":
        return valor * 0.9 // 10% de desconto
    case "vip-platina":
        return valor * 0.8 // 20% de desconto
    default:
        return valor
    }
}
```

Cada tipo de cliente novo obriga a reabrir essa função e adicionar mais um `case`. Isso não é catastrófico com 3 casos, mas em sistemas reais esse `switch` costuma crescer para dezenas de casos, espalhados às vezes por várias funções parecidas — e toda alteração aqui é risco de quebrar um `case` que não tinha nada a ver com a mudança.

### Versão refatorada

```go
package desconto

// PoliticaDesconto é a abstração: qualquer tipo que souber calcular um
// desconto satisfaz essa interface, sem precisar avisar ninguém disso.
type PoliticaDesconto interface {
    Aplicar(valor float64) float64
}

type SemDesconto struct{}

func (SemDesconto) Aplicar(valor float64) float64 { return valor }

type DescontoVIP struct{}

func (DescontoVIP) Aplicar(valor float64) float64 { return valor * 0.9 }

type DescontoVIPPlatina struct{}

func (DescontoVIPPlatina) Aplicar(valor float64) float64 { return valor * 0.8 }

// CalcularDesconto nunca mais precisa mudar — ela só usa a política que
// recebeu, seja qual for.
func CalcularDesconto(p PoliticaDesconto, valor float64) float64 {
    return p.Aplicar(valor)
}
```

Quando aparecer um quinto tipo de cliente amanhã, a resposta é: cria um `struct` novo com um método `Aplicar`, e pronto — `CalcularDesconto` está "fechada para modificação" (ninguém mexe nela de novo) mas "aberta para extensão" (comportamento novo é só um tipo novo ao lado). Repare que isso só foi possível porque a interface `PoliticaDesconto` existe — é o mesmo mecanismo de interface implícita que sustenta praticamente todo SOLID em Go.

## L — Liskov Substitution Principle (Princípio da Substituição de Liskov)

**Em palavras simples**: se um pedaço de código espera algo que satisfaça uma interface, qualquer implementação concreta dessa interface deveria poder ser usada no lugar de outra sem quebrar o comportamento esperado. Não basta ter os métodos com a assinatura certa (isso o compilador já garante) — o *comportamento* também precisa respeitar o que a interface promete implicitamente.

Esse princípio nasceu pensando em "subclasse substitui superclasse", mas como Go não tem classe, a tradução natural é: "qualquer implementação de uma interface substitui qualquer outra implementação da mesma interface, sem surpresa".

### Versão ingênua

```go
package notificacao

import "errors"

// Notifier é o contrato: qualquer implementação deve tentar notificar e,
// se falhar, retornar um erro — nunca travar o programa.
type Notifier interface {
    Notify(mensagem string) error
}

type EmailNotifier struct{}

func (EmailNotifier) Notify(mensagem string) error {
    // simulação: manda e-mail, se der erro, retorna um error normalmente
    return nil
}

// SMSNotifier "satisfaz" a interface (o compilador aceita), mas quebra o
// contrato implícito: em vez de devolver um erro quando falha, ela entra
// em pânico (panic é explicado no tópico Tratamento de erros e pacotes).
// Qualquer código escrito esperando "erro tratável" quebra na hora que
// alguém trocar EmailNotifier por SMSNotifier.
type SMSNotifier struct{}

func (SMSNotifier) Notify(mensagem string) error {
    if mensagem == "" {
        panic("mensagem vazia") // deveria ser: return errors.New("mensagem vazia")
    }
    return nil
}
```

O código que usa `Notifier` provavelmente foi escrito assumindo "se der problema, eu recebo um `error` e trato ali". `SMSNotifier` quebra essa expectativa silenciosamente: o programa inteiro derruba (`panic`) em vez de devolver o erro esperado. Isso compila sem nenhum aviso — o compilador só olha a assinatura do método, nunca o comportamento — e só aparece como bug quando alguém trocar de implementação em produção.

### Versão refatorada

```go
package notificacao

import "errors"

type SMSNotifier struct{}

func (SMSNotifier) Notify(mensagem string) error {
    if mensagem == "" {
        return errors.New("mensagem vazia") // agora respeita o contrato
    }
    return nil
}
```

Um segundo exemplo real e muito citado desse mesmo problema é a interface `Read` da biblioteca padrão do Go (`io.Reader`): o contrato implícito diz que, quando os dados acabam, a implementação deve devolver o erro especial `io.EOF` — não qualquer outro erro genérico. Uma implementação que devolve um erro diferente nesse caso "satisfaz" a interface (compila) mas quebra qualquer código escrito esperando `io.EOF` especificamente, porque laços de leitura em Go costumam checar exatamente esse valor pra saber quando parar. O exemplo completo está em [go/config/reference/SOLID.md](../config/reference/SOLID.md#l--liskov-substitution).

O ponto prático pra levar daqui: ao escrever uma implementação nova de uma interface já existente, leia (ou pergunte) qual é o comportamento que as outras implementações já garantem, além da assinatura — isso normalmente não está escrito em lugar nenhum além do próprio código das outras implementações e, se tiver sorte, no doc comment da interface.

## I — Interface Segregation Principle (Princípio da Segregação de Interfaces)

**Em palavras simples**: é melhor ter várias interfaces pequenas e específicas do que uma única interface grande e genérica. Se uma interface tem 6 métodos e cada tipo que a implementa só usa 2 deles de verdade, isso é sinal de que ela deveria ser quebrada em interfaces menores.

### Versão ingênua

```go
package funcionario

// Funcionario junta métodos de áreas bem diferentes: cálculo de salário,
// controle de ponto, admissão e demissão. Qualquer tipo que precise
// implementar essa interface é obrigado a implementar TODOS os métodos,
// mesmo que só use um ou dois.
type Funcionario interface {
    CalcularSalario() float64
    BaterPonto() error
    Contratar(nome string) error
    Demitir(motivo string) error
    GerarFolhaDePagamento() []byte
}
```

Imagine um relatório simples que só precisa saber o salário de cada funcionário pra somar um total. Se a função que gera esse relatório recebe um parâmetro do tipo `Funcionario`, ela está "carregando" a obrigação de existirem `Contratar` e `Demitir` — métodos que ela nunca vai chamar, mas que tornam qualquer implementação de teste (um "funcionário falso" pra testar o relatório) muito mais trabalhosa de escrever, porque você é obrigado a implementar métodos que não interessam pro teste.

### Versão refatorada

```go
package funcionario

// Cada interface tem um único propósito e só os métodos que esse
// propósito realmente exige.
type CalculadoraSalario interface {
    CalcularSalario() float64
}

type RegistradorPonto interface {
    BaterPonto() error
}

type GestorContratacao interface {
    Contratar(nome string) error
    Demitir(motivo string) error
}

// O relatório de folha de pagamento só precisa saber calcular salário —
// então é só isso que ele pede como dependência.
func TotalFolha(funcionarios []CalculadoraSalario) float64 {
    total := 0.0
    for _, f := range funcionarios {
        total += f.CalcularSalario()
    }
    return total
}
```

Um `struct` real de funcionário ainda pode implementar todos os métodos de todas essas interfaces pequenas ao mesmo tempo — nada te impede disso. A diferença é que quem *consome* (a função `TotalFolha`) agora pede explicitamente só o pedaço mínimo de que precisa (`CalculadoraSalario`), em vez do pacote inteiro (`Funcionario`). Isso deixa o teste de `TotalFolha` muito mais simples (um "funcionário falso" pra teste só precisa implementar um método) e deixa claro, só lendo a assinatura da função, exatamente do que ela depende.

## D — Dependency Inversion Principle (Princípio da Inversão de Dependência)

**Em palavras simples**: código de "alto nível" (a regra de negócio) não deveria depender diretamente de código de "baixo nível" (detalhes técnicos, como qual banco de dados é usado). Os dois deveriam depender de uma abstração (uma interface) — e essa abstração deveria "pertencer" a quem consome (o código de alto nível), não a quem implementa (o código de baixo nível).

### Versão ingênua

```go
package pedido

import "database/sql"

// PedidoService depende diretamente de *sql.DB — um detalhe concreto de
// infraestrutura (qual banco, como conectar, SQL específico). Se amanhã
// o time decidir trocar de Postgres pra outro banco, ou usar um serviço
// de fila em vez de banco direto, esse código de regra de negócio
// precisa ser reescrito, mesmo que a REGRA em si não tenha mudado nada.
type PedidoService struct {
    db *sql.DB
}

func NewPedidoService(db *sql.DB) *PedidoService {
    return &PedidoService{db: db}
}

func (s *PedidoService) Cancelar(id string) error {
    _, err := s.db.Exec("UPDATE pedidos SET status = 'cancelado' WHERE id = ?", id)
    return err
}
```

Testar `Cancelar` aqui exige um banco de verdade rodando — não dá pra testar a regra "não deveria dar pra cancelar um pedido já enviado" (que nem existe ainda neste código!) sem também levantar infraestrutura de banco.

### Versão refatorada

```go
package pedido

// PedidoRepository é a abstração — pertence a este pacote (o "alto
// nível"), não ao pacote que vai implementar o banco de verdade.
// Isso é a inversão: em vez do domínio depender do banco, o banco é
// quem vai depender (implementar) desta interface.
type PedidoRepository interface {
    AtualizarStatus(id string, status string) error
}

type PedidoService struct {
    repo PedidoRepository // depende da abstração, não de *sql.DB
}

func NewPedidoService(repo PedidoRepository) *PedidoService {
    return &PedidoService{repo: repo}
}

func (s *PedidoService) Cancelar(id string) error {
    return s.repo.AtualizarStatus(id, "cancelado")
}
```

```go
// Em outro pacote, separado — só aqui entra database/sql de verdade.
package postgres

import "database/sql"

type PedidoRepo struct{ db *sql.DB }

func (r *PedidoRepo) AtualizarStatus(id, status string) error {
    _, err := r.db.Exec("UPDATE pedidos SET status = ? WHERE id = ?", status, id)
    return err
}
```

Agora dá pra testar `PedidoService.Cancelar` com uma implementação falsa de `PedidoRepository` (um `struct` simples que só guarda o último status recebido em memória, sem banco nenhum). E se o time decidir usar uma API HTTP em vez de banco direto, só precisa escrever um novo tipo que implemente `PedidoRepository` — `PedidoService` não muda uma linha. Repare que essa mesma ideia é o coração do que Clean Architecture chama de "regra de dependência" (ver [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/)) — dependência de código sempre aponta para dentro, na direção da regra de negócio, nunca o contrário.

Uma observação sobre como montar isso na prática: Go não tem (nem costuma usar) um framework de "injeção de dependência" automático. A montagem é manual — normalmente dentro da função `main()` do programa, que é o único lugar que conhece tanto a interface quanto a implementação concreta:

```go
func main() {
    db, _ := sql.Open("postgres", "...")
    repo := &postgres.PedidoRepo{db: db}
    service := pedido.NewPedidoService(repo) // aqui a "injeção" acontece
    // ...
}
```

## Como os 5 se combinam num pacote pequeno

Na prática, os 5 princípios costumam aparecer juntos, reforçando um ao outro, dentro de uma organização de arquivos como esta:

```
pedido/
  pedido.go        // struct Pedido + método Cancelar() — SRP: só sabe validar/mudar a si mesmo
  repository.go     // interface PedidoRepository — ISP (poucos métodos) + DIP (abstração perto de quem usa)
  service.go          // PedidoService usa PedidoRepository via injeção de construtor — OCP: novo repo = nova implementação, service não muda
```

O padrão que se repete: **struct pequeno e focado (S)**, **dependência declarada como interface pequena que o consumidor define (I + D)**, **novo comportamento = novo tipo satisfazendo a interface, sem editar o que já existe (O)**. O tópico [Repository pattern na prática](../repository-pattern-na-pratica/) (Módulo 3) mostra essa mesma ideia com um exemplo mais completo, incluindo implementação real de banco.

## Erros comuns de quem está começando em Go

- **Recriar hierarquia de tipo via embedding**: usar embedding tentando simular herança (`Dog` embute `Animal` só pra reaproveitar código, não porque `Dog` "é um" `Animal` de verdade). Embedding é composição — se o motivo real é só reaproveitar um método, prefira um campo nomeado comum + chamar o método explicitamente, ou repensar se a abstração faz sentido.
- **Interface grande "genérica" definida antes de precisar**: desenhar o contrato inteiro, com todos os métodos que "algum dia" podem ser necessários, antes de existir um segundo caso de uso real que use aquilo. Em Go, o costume idiomático é esperar o segundo consumidor aparecer para então extrair a interface — uma interface com 1 implementação e nenhuma alternativa em vista é abstração prematura (ver também a seção de code smells em [Clean Code em Go](../clean-code-em-go/)).
- **Injeção de dependência via framework por hábito**: trazer uma biblioteca externa de "container de injeção de dependência" pra Go quando o projeto é pequeno. O padrão idiomático de Go é injetar manualmente, via construtor (`NewService(repo Repository)`), montado à mão dentro de `main.go` — isso já é suficiente na esmagadora maioria dos serviços Go, e trazer um framework de DI geralmente é over-engineering pro tamanho típico de um serviço nessa linguagem.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise solid em go`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
