# Structs, métodos e interfaces

## Structs — dado puro, sem método embutido

Em Python, `class` já junta dado e comportamento na mesma declaração. Em Go, `struct` é só dado:

```go
type Order struct {
    ID     string
    Amount float64
}
```

Método é uma função separada, ligada ao tipo via **receiver**:

```go
func (o Order) Total() float64 {
    return o.Amount
}
```

`(o Order)` é o receiver — o equivalente ao `self` implícito do Python, mas explícito e escolhido por você a cada método.

## Receiver por valor vs por ponteiro

Essa escolha não existe em Python (lá, tudo é referência) e é uma decisão real em Go:

```go
func (o Order) Total() float64 { return o.Amount }        // recebe cópia — não pode mutar o Order original
func (o *Order) SetAmount(v float64) { o.Amount = v }      // recebe ponteiro — muta o Order original
```

Regra prática: se o método muda o struct, ou se o struct é grande (copiar sai caro), use receiver por ponteiro (`*Order`). Se é só leitura e o struct é pequeno, valor está ok. Inconsistência (misturar os dois no mesmo tipo) é code smell.

## Sem herança — composição via embedding

Go não tem `class Filho(Pai)`. O equivalente é **embedding**: colocar um struct dentro de outro sem nome de campo, o que "promove" os campos/métodos automaticamente:

```go
type Animal struct{ Name string }
func (a Animal) Describe() string { return "sou " + a.Name }

type Dog struct {
    Animal   // embedding — não "herança", Dog TEM um Animal
    Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Vira-lata"}
d.Describe()   // funciona: método promovido do Animal embutido
```

Isso é composição, não herança — não existe polimorfismo por override automático de método como em Python. Se `Dog` quiser um `Describe()` próprio, ele simplesmente declara o método com o mesmo nome, e o do `Animal` embutido fica "sombreado", não substituído por despacho dinâmico.

## Interfaces — satisfação implícita

Em Python você normalmente declara explicitamente `class Foo(Protocol)` ou `class Foo(ABC)`. Em Go, um tipo satisfaz uma interface automaticamente, só por ter os métodos — sem `implements`, sem declarar a relação em lugar nenhum:

```go
type Shape interface {
    Area() float64
}

type Circle struct{ Radius float64 }
func (c Circle) Area() float64 { return math.Pi * c.Radius * c.Radius }

// Circle satisfaz Shape automaticamente — nenhuma linha de código declara isso.
var s Shape = Circle{Radius: 2}
```

Isso é o que viabiliza Interface Segregation e Dependency Inversion do jeito que Go faz — ver exemplo em [go/config/reference/SOLID.md](../config/reference/SOLID.md#i--interface-segregation) e [contexts/common/SOLID.md](../../contexts/common/SOLID.md). Como não há vínculo explícito, a interface pode (e deve) ser declarada perto de quem consome, não de quem implementa — um pacote nem precisa saber que sua struct será usada como uma interface definida em outro lugar.

## Struct zero value já é utilizável

Retomando o zero value do tópico anterior: `var o Order` já é um `Order{ID: "", Amount: 0}` válido, sem precisar de construtor. Construtor (`NewOrder(...)`) em Go é convenção, não exigência da linguagem — só cria quando há inicialização não-trivial a fazer.
