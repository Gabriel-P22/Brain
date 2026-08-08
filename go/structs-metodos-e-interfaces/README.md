# Structs, métodos e interfaces

## Struct — um tipo que agrupa vários dados relacionados

Até agora você viu tipos simples: `int`, `string`, `bool`. Mas a maioria dos problemas reais envolve dados que fazem sentido juntos — por exemplo, um pedido de compra tem um identificador, um valor e um status. Você poderia guardar isso em três variáveis soltas, mas aí nada impede que elas se desalinhem (mudar o ID de um pedido sem mudar o valor junto, por engano).

**Struct** é a forma de Go de criar um tipo novo que agrupa vários campos nomeados, cada um com seu próprio tipo, como uma unidade só:

```go
package main

import "fmt"

type Order struct {
    ID     string
    Amount float64
    Status string
}

func main() {
    o := Order{
        ID:     "abc-123",
        Amount: 150.00,
        Status: "pendente",
    }

    fmt.Println(o.ID)     // abc-123
    fmt.Println(o.Amount) // 150
    o.Status = "pago"     // acessa e altera um campo com ponto
    fmt.Println(o.Status) // pago
}
```

Repare na sintaxe: `type Order struct { ... }` declara um tipo novo chamado `Order`. Dentro das chaves, cada linha é um campo: um nome (`ID`, `Amount`, `Status`) seguido do tipo daquele campo. Depois de declarado, `Order` passa a ser um tipo normal, igual `int` ou `string` — dá para declarar variáveis dele, passá-lo como argumento de função, colocá-lo dentro de outro struct, etc.

Para criar um valor desse tipo, você usa um **struct literal** — o nome do tipo seguido de chaves com os valores de cada campo. Duas formas comuns:

```go
o1 := Order{ID: "abc-123", Amount: 150.00, Status: "pendente"} // com nome de campo — recomendado
o2 := Order{"abc-123", 150.00, "pendente"}                      // por posição — funciona, mas frágil:
                                                                    // se a ordem dos campos no struct mudar, o código quebra silenciosamente
```

A forma com nome de campo (`ID: "..."`) é a idiomática (o jeito "correto"/esperado de escrever em Go) porque não depende da ordem em que os campos foram declarados no `type`, e deixa claro pra quem lê o código qual valor vai em qual campo.

Um struct pode conter outro struct como campo, o que é a base de como Go representa dados aninhados (um pedido que tem um endereço de entrega, por exemplo):

```go
type Address struct {
    Street string
    City   string
}

type Order struct {
    ID      string
    Amount  float64
    Deliver Address // struct dentro de struct — campo comum, com nome
}

o := Order{
    ID:     "abc-123",
    Amount: 150.00,
    Deliver: Address{
        Street: "Rua das Flores, 100",
        City:   "São Paulo",
    },
}
fmt.Println(o.Deliver.City) // São Paulo — acesso encadeado com ponto
```

## Método — uma função ligada a um tipo

Um struct, sozinho, só guarda dado — não tem comportamento embutido nele. Para dar comportamento a um tipo, você declara uma função separada e a "liga" a esse tipo através de algo chamado **receiver** (receptor):

```go
type Order struct {
    ID     string
    Amount float64
}

func (o Order) Description() string {
    return fmt.Sprintf("pedido %s no valor de R$%.2f", o.ID, o.Amount)
}
```

A diferença entre isso e uma função comum está só na parte `(o Order)` logo depois de `func`. Isso se chama receiver: diz "esta função pode ser chamada como se fosse um método do tipo `Order`, e dentro dela, `o` é a variável que representa o valor de `Order` sobre o qual o método foi chamado". Depois de declarado assim, você chama o método com ponto, como se fosse um campo:

```go
o := Order{ID: "abc-123", Amount: 150.00}
fmt.Println(o.Description()) // pedido abc-123 no valor de R$150.00
```

Repare que `o` (o nome do receiver) é uma escolha sua — poderia se chamar qualquer coisa (`func (this Order) ...`, `func (order Order) ...`). A convenção idiomática em Go é usar uma ou duas letras curtas, geralmente a primeira letra do tipo em minúsculo (`o` para `Order`, `u` para `User`), não um nome genérico como `this` ou `self`.

## Receiver por valor vs. por ponteiro

Esta é uma decisão real que você precisa tomar toda vez que declara um método, e ela afeta o comportamento do programa. Existem dois jeitos de escrever um receiver:

```go
func (o Order) Total() float64 { return o.Amount }         // receiver por VALOR
func (o *Order) SetAmount(v float64) { o.Amount = v }       // receiver por PONTEIRO
```

- **Receiver por valor** (`o Order`): quando o método é chamado, Go faz uma **cópia** inteira do struct e entrega essa cópia para dentro do método. Qualquer alteração feita em `o` dentro do método acontece só nessa cópia — o struct original, de quem chamou o método, não é afetado.
- **Receiver por ponteiro** (`o *Order`): em vez de uma cópia, o método recebe o **endereço de memória** de onde o struct original está guardado (ponteiro — assunto explicado em detalhe no próximo tópico, [Ponteiros, slices e maps](../ponteiros-slices-e-maps/)). Alterações feitas em `o` dentro do método afetam o struct original de verdade.

Veja a diferença de comportamento lado a lado:

```go
package main

import "fmt"

type Order struct{ Amount float64 }

func (o Order) TryChangeByValue(v float64) {
    o.Amount = v // só muda a cópia local
}

func (o *Order) ChangeByPointer(v float64) {
    o.Amount = v // muda o struct original
}

func main() {
    o := Order{Amount: 100}

    o.TryChangeByValue(999)
    fmt.Println(o.Amount) // ainda 100 — a mudança não "vazou" pra fora do método

    o.ChangeByPointer(999)
    fmt.Println(o.Amount) // 999 — agora mudou de verdade
}
```

Regra prática para decidir qual usar:

- Se o método **precisa alterar** algum campo do struct, use receiver por ponteiro (`*Order`) — é a única forma de a mudança realmente acontecer no valor original.
- Se o struct é **grande** (muitos campos, ou campos pesados), prefira ponteiro mesmo só para leitura, porque copiar um struct grande a cada chamada tem custo de memória e tempo.
- Se o método só lê dados e o struct é pequeno (poucos campos simples), receiver por valor é perfeitamente aceitável.
- **Não misture os dois no mesmo tipo sem motivo** — se `Order` tem um método com `*Order` e outro com `Order`, isso é considerado um sinal de código malfeito (code smell): a convenção idiomática é escolher um estilo por tipo e manter consistente em todos os métodos daquele tipo.

## Composição — um struct "tem" outro, em vez de "ser" outro

Muitos problemas de organização de código podem ser resolvidos reaproveitando comportamento entre tipos parecidos. Go resolve isso através de **composição**: um struct pode conter outro struct dentro de si, e opcionalmente "herdar" seus campos e métodos de um jeito automático chamado **embedding** (quando o campo aninhado não recebe um nome próprio):

```go
package main

import "fmt"

type Animal struct {
    Name string
}

func (a Animal) Describe() string {
    return "eu sou " + a.Name
}

type Dog struct {
    Animal // embedding: sem nome de campo — isso "promove" os campos/métodos de Animal
    Breed  string
}

func main() {
    d := Dog{
        Animal: Animal{Name: "Rex"},
        Breed:  "Vira-lata",
    }

    fmt.Println(d.Name)        // Rex — campo de Animal acessado direto em Dog, sem precisar escrever d.Animal.Name
    fmt.Println(d.Describe())  // eu sou Rex — método de Animal chamado direto em Dog
    fmt.Println(d.Breed)       // Vira-lata — campo próprio de Dog
}
```

O ponto importante aqui: isso é **composição** ("`Dog` TEM um `Animal` dentro dele"), não um mecanismo de herança de classe ("`Dog` É UM tipo de `Animal` que herda e pode sobrescrever comportamento com despacho dinâmico"). Go não tem esse segundo conceito. A consequência prática: se `Dog` declarar seu próprio método `Describe()`, ele simplesmente **esconde** (não "sobrescreve" no sentido polimórfico) o `Describe()` de `Animal` quando chamado através de uma variável do tipo `Dog` — não existe uma tabela de despacho dinâmico decidindo em tempo de execução qual versão rodar.

```go
func (d Dog) Describe() string {
    return "eu sou " + d.Name + ", um cachorro da raça " + d.Breed
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Vira-lata"}
fmt.Println(d.Describe()) // usa o Describe de Dog, não o de Animal — método próprio tem prioridade
```

Esse jeito de organizar código — favorecer composição (montar comportamento juntando peças pequenas) em vez de uma hierarquia de herança — é um princípio de design conhecido como "composição sobre herança". Ele evita um problema comum de hierarquias profundas: mudar uma classe "avó" tendo efeito colateral inesperado em todas as "netas", sem ninguém perceber olhando só pro código que mudou.

## Interfaces — um contrato de comportamento

Até aqui, tudo o que vimos amarra um tipo concreto (`Order`, `Dog`) a um comportamento específico. Mas muitas vezes você quer escrever código que funciona com **qualquer tipo que saiba fazer uma certa coisa**, sem se importar com qual tipo exatamente é esse. É pra isso que existe **interface**.

Uma interface declara um conjunto de métodos (a "assinatura" de cada um: nome, parâmetros, retorno) sem implementar nenhum deles — é só uma lista do que um tipo precisa saber fazer para "servir" como aquela interface:

```go
package main

import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func printArea(s Shape) {
    fmt.Printf("área: %.2f\n", s.Area())
}

func main() {
    printArea(Circle{Radius: 2})       // área: 12.57
    printArea(Rectangle{Width: 3, Height: 4}) // área: 12.00
}
```

O detalhe mais importante deste exemplo inteiro: **em nenhum lugar do código existe uma linha dizendo "`Circle` implementa `Shape`"**. Não existe uma palavra-chave `implements`, nem nenhuma declaração de que esses tipos estão relacionados. `Circle` e `Rectangle` simplesmente têm um método `Area() float64` cada um — e isso, sozinho, já é suficiente para que ambos possam ser usados em qualquer lugar que espera um `Shape`. Isso se chama **satisfação implícita de interface** (também descrita como "tipagem estrutural"): o compilador checa, no momento de compilar, se o tipo tem os métodos exigidos — a relação "esse tipo satisfaz essa interface" é descoberta automaticamente, não declarada manualmente.

Um outro tipo pode satisfazer a mesma interface `Shape` sem nem saber que `Shape` existe — não precisa nem estar no mesmo pacote:

```go
type Square struct {
    Side float64
}

func (s Square) Area() float64 {
    return s.Side * s.Side
}

// Square também satisfaz Shape automaticamente, mesmo declarado bem depois,
// mesmo que quem escreveu Square nunca tenha visto a definição de Shape.
printArea(Square{Side: 5}) // área: 25.00
```

### Por que isso importa para design de software

Essa satisfação implícita é o mecanismo concreto que Go usa para viabilizar dois dos princípios SOLID (definição completa em [contexts/common/SOLID.md](../../contexts/common/SOLID.md), exemplos de código Go em [go/config/reference/SOLID.md](../config/reference/SOLID.md)):

- **Interface Segregation** (o "I" de SOLID): como não existe custo nenhum em declarar uma interface nova (nenhuma implementação precisa "assinar embaixo" dela), o idiomático em Go é ter várias interfaces pequenas e específicas (frequentemente com um único método) em vez de uma interface grande genérica. Veja o exemplo em [go/config/reference/SOLID.md](../config/reference/SOLID.md#i--interface-segregation).
- **Dependency Inversion** (o "D" de SOLID): como não há vínculo explícito entre quem implementa e a interface, a interface pode (e deve, pelo idiomático de Go) ser declarada **perto de quem consome** ela, não perto de quem implementa. Isso significa que um pacote que faz a regra de negócio pode declarar "eu preciso de algo que salve um pedido" (uma interface pequena) sem nunca importar o pacote que sabe conversar com o banco de dados de verdade — é esse pacote de infraestrutura que, ao implementar os métodos certos, passa a satisfazer a interface. Veja em [go/config/reference/SOLID.md](../config/reference/SOLID.md#d--dependency-inversion).

Uma variável do tipo de uma interface pode guardar qualquer valor concreto que satisfaça aquela interface:

```go
var s Shape = Circle{Radius: 2} // s é do tipo Shape, mas guarda um Circle por baixo
s = Rectangle{Width: 3, Height: 4} // agora guarda um Rectangle — troca de implementação sem mudar o tipo da variável
```

## Struct zero value: já nasce pronto pra uso

Retomando o conceito de **zero value** do tópico [Sintaxe básica](../sintaxe-basica/#zero-value-toda-variável-já-nasce-com-um-valor-utilizável): declarar um struct sem inicializar nenhum campo não deixa ele "vazio" ou inválido — cada campo automaticamente recebe o zero value do próprio tipo dele:

```go
type Order struct {
    ID     string
    Amount float64
    Paid   bool
}

var o Order
fmt.Println(o) // {  0 false} — ID é "" (string vazia), Amount é 0, Paid é false
fmt.Println(o.ID == "") // true
```

Isso significa que, diferente de precisar sempre escrever um construtor explícito antes de usar um tipo, em Go `var o Order` já é um valor pronto pra uso imediato. Uma função construtora (por convenção chamada `NewOrder(...)`, começando com `New`) não é uma exigência da linguagem — ela só é necessária quando existe alguma inicialização não-trivial a fazer (validar um argumento, calcular um campo derivado, garantir que um campo obrigatório nunca fique vazio):

```go
func NewOrder(id string, amount float64) (Order, error) {
    if id == "" {
        return Order{}, fmt.Errorf("id do pedido não pode ser vazio")
    }
    return Order{ID: id, Amount: amount}, nil
}
```

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise structs, métodos e interfaces`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
