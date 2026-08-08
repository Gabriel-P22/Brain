# Tratamento de erros e pacotes

## error é só uma interface — não um mecanismo especial da linguagem

Você já viu, desde o tópico [Sintaxe básica](../sintaxe-basica/#funções-com-múltiplos-retornos), funções que devolvem um resultado junto com um `error`:

```go
resultado, err := dividir(10, 0)
if err != nil {
    fmt.Println("deu erro:", err)
    return
}
```

O que talvez não estivesse claro ainda: `error` não é uma construção mágica embutida no compilador — é apenas uma **interface**, exatamente do mesmo tipo de coisa que você viu em [Structs, métodos e interfaces](../structs-metodos-e-interfaces/#interfaces--um-contrato-de-comportamento). A definição inteira de `error`, presente na biblioteca padrão do Go, é esta:

```go
type error interface {
    Error() string
}
```

Ou seja: **qualquer tipo que tenha um método `Error() string` já é, automaticamente, um erro válido** — pela mesma regra de satisfação implícita que você já conhece. Isso quer dizer que a forma mais simples de criar um erro é usar a função pronta `errors.New`, que devolve um valor de um tipo interno que só guarda uma mensagem de texto:

```go
package main

import (
    "errors"
    "fmt"
)

func dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divisão por zero")
    }
    return a / b, nil
}

func main() {
    resultado, err := dividir(10, 0)
    if err != nil {
        fmt.Println("erro:", err) // erro: divisão por zero
        return
    }
    fmt.Println(resultado)
}
```

Quando `b == 0`, a função devolve `0` (o zero value de `float64`, já que não há resultado válido nesse caso) e um `error` não-nil. Quando não há erro, a convenção idiomática é sempre devolver `nil` no lugar do `error` — e `if err != nil` é o padrão mais comum que você vai ver em qualquer código Go real, exatamente porque devolver erro como um valor comum (em vez de interromper o fluxo do programa) obriga quem chama a função a decidir explicitamente o que fazer quando algo dá errado.

### Erro customizado — quando uma mensagem de texto não é suficiente

Porque `error` é só uma interface, você pode criar seu próprio tipo de erro, com dados estruturados além do texto — útil quando quem trata o erro precisa examinar detalhes dele, não só imprimir uma mensagem:

```go
package main

import "fmt"

type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s com id %s não encontrado", e.Resource, e.ID)
}

func buscarUsuario(id string) (string, error) {
    if id != "42" {
        return "", &NotFoundError{Resource: "usuário", ID: id}
    }
    return "Ana", nil
}

func main() {
    _, err := buscarUsuario("99")
    if err != nil {
        fmt.Println(err) // usuário com id 99 não encontrado
    }
}
```

Repare que `Error()` foi declarado com receiver por ponteiro (`*NotFoundError`, ver [Structs, métodos e interfaces](../structs-metodos-e-interfaces/#receiver-por-valor-vs-por-ponteiro)), e por isso a função devolve `&NotFoundError{...}` (o endereço da struct, não a struct em si) — é `*NotFoundError` que satisfaz a interface `error` aqui, não `NotFoundError` puro. Mais adiante nesta seção você vai ver como recuperar esse tipo customizado de volta a partir de um `error` genérico, usando `errors.As`.

## Wrapping — adicionando contexto a um erro conforme ele sobe na cadeia de chamadas

Go não empilha automaticamente nenhum histórico de "por onde esse erro passou" enquanto ele sobe de função em função (diferente de mecanismos de exceção de outras linguagens, que costumam montar esse histórico sozinhos). Se você simplesmente devolver um `error` sem adicionar nada, quem receber lá no topo só vai ver a mensagem original, sem saber em qual ponto do programa ela aconteceu. O idiomático em Go é **envolver** (wrap, "embrulhar") o erro com contexto extra, a cada camada que ele atravessa:

```go
package main

import (
    "database/sql"
    "fmt"
)

func loadUser(id string) (string, error) {
    nome, err := buscarNoBanco(id)
    if err != nil {
        return "", fmt.Errorf("carregando usuário %s: %w", id, err)
    }
    return nome, nil
}

func buscarNoBanco(id string) (string, error) {
    // simulação de erro vindo de uma chamada de banco de dados
    return "", sql.ErrNoRows
}

func main() {
    _, err := loadUser("42")
    fmt.Println(err) // carregando usuário 42: sql: no rows in result set
}
```

O verbo `%w` dentro de `fmt.Errorf` é diferente de `%v` ou `%s`: ele não apenas coloca o texto do erro original na mensagem nova, ele **preserva uma referência ao erro original**, encadeado dentro do erro novo. Isso permite que código mais acima na cadeia de chamadas consiga "desembrulhar" e examinar o erro original, mesmo depois de várias camadas de wrapping — duas funções da biblioteca padrão fazem exatamente isso:

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
)

func main() {
    err := fmt.Errorf("carregando usuário 42: %w", sql.ErrNoRows)

    // errors.Is: "esse erro é (ou embrulha, em algum nível) ESTE erro específico?"
    if errors.Is(err, sql.ErrNoRows) {
        fmt.Println("usuário não existe no banco")
    }

    // errors.As: "esse erro é (ou embrulha) um erro DESTE TIPO? se for, me dá acesso a ele"
    var nf *NotFoundError
    if errors.As(err, &nf) {
        fmt.Println("é um NotFoundError:", nf.Resource)
    }
}

type NotFoundError struct{ Resource, ID string }

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %s não encontrado", e.Resource, e.ID)
}
```

A diferença entre as duas funções:

- `errors.Is(err, alvo)` — usada quando você quer comparar contra um **valor específico já conhecido** de erro (chamado de "sentinel error" — um erro pré-criado e exportado, como `sql.ErrNoRows`, especificamente para ser comparado assim). Pergunta: "esse erro é exatamente este aqui, em algum nível do wrapping?".
- `errors.As(err, &variavel)` — usada quando você quer checar se o erro é (ou embrulha) um **tipo customizado específico**, e, se for, obter acesso aos campos daquele tipo. Pergunta: "esse erro tem, em algum nível do wrapping, um erro deste tipo aqui? se sim, coloca ele nessa variável".

Ambas percorrem toda a cadeia de erros embrulhados (usando `%w` repetidamente) até encontrar uma correspondência ou chegar ao fim — você não precisa desembrulhar manualmente camada por camada.

## panic e recover — não é o mesmo mecanismo de erro esperado

Go tem uma segunda forma de interromper o fluxo normal de execução, chamada `panic`. É importante entender que `panic` **não é o jeito de tratar erro do dia a dia** em Go — arquivo não encontrado, validação de formulário que falhou, usuário não existe: tudo isso é sempre representado como um `error` retornado normalmente, do jeito que você já viu acima.

`panic` é reservado para situações de **estado realmente inconsistente do programa** — bugs de programação, não condições esperadas de negócio. Os casos mais comuns de panic acontecem sozinhos, sem você chamar `panic` explicitamente:

```go
package main

func main() {
    var s []int
    _ = s[5] // panic: runtime error: index out of range [5] with length 0

    var p *int
    _ = *p // panic: runtime error: invalid memory address or nil pointer dereference
}
```

Você também pode chamar `panic` manualmente, mas isso deveria ser raro e reservado para situações onde continuar rodando o programa seria pior do que parar imediatamente (por exemplo, uma configuração obrigatória que está ausente na inicialização do programa, antes de qualquer requisição chegar a ser atendida):

```go
func mustLoadConfig() Config {
    cfg, err := loadConfig()
    if err != nil {
        panic("configuração obrigatória não encontrada: " + err.Error())
    }
    return cfg
}
```

Por padrão, um `panic` não tratado derruba o programa inteiro (imprime uma pilha de chamadas e encerra o processo). `recover()` existe para os casos em que você precisa **evitar que um panic derrube todo o processo** — mas só funciona quando chamado de dentro de uma função adiada com `defer` (assunto abordado com mais profundidade no tópico de concorrência):

```go
package main

import "fmt"

func rodarComSeguranca(f func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recuperado de um panic:", r)
        }
    }()
    f()
}

func main() {
    rodarComSeguranca(func() {
        var s []int
        _ = s[5] // isso panica...
    })
    fmt.Println("...mas o programa continua rodando até aqui") // esta linha ainda executa
}
```

Usar `recover` como forma geral de tratar erro esperado (por exemplo, dar `panic` de propósito toda vez que uma validação falha, e usar `recover` pra "capturar" isso) é considerado **anti-idiomático** em Go — foge do padrão que toda a comunidade e toda a biblioteca padrão seguem, que é `error` como retorno explícito. O uso legítimo mais comum de `recover` em código de produção é em um único lugar centralizado — um middleware de servidor HTTP, por exemplo, que protege o processo inteiro contra um bug não previsto em qualquer requisição, sem deixar aquele bug derrubar o servidor pra todo mundo. Isso é revisitado com um exemplo completo no Módulo 3.

## Pacotes — a unidade de organização de código em Go

Em Go, **um pacote corresponde a uma pasta**. Todo arquivo `.go` dentro de uma pasta declara, na primeira linha, a qual pacote ele pertence — e todos os arquivos `.go` de uma mesma pasta precisam declarar o mesmo pacote:

```go
// arquivo: payment/processor.go
package payment

func Process(amount float64) error {
    // ...
    return nil
}
```

```go
// arquivo: payment/validator.go — mesma pasta, mesmo pacote
package payment

func Validate(amount float64) error {
    // ...
    return nil
}
```

Não existe arquivo Go "solto" fora de um pacote — todo arquivo `.go` precisa começar com uma linha `package algumnome`. Outro código que queira usar as funções de `payment` faz isso importando o caminho da pasta:

```go
package main

import "meumodulo/payment"

func main() {
    err := payment.Process(150.00)
    _ = err
}
```

### Visibilidade: a primeira letra decide o que é público

Go não tem uma palavra-chave como `private` ou `public`. Em vez disso, a **visibilidade de um identificador (função, tipo, variável, campo de struct) é determinada só pela primeira letra do nome dele**:

- Começa com **letra maiúscula** (`PascalCase`) → **exportado**: visível e utilizável por qualquer outro pacote que importar este.
- Começa com **letra minúscula** (`camelCase`) → **não-exportado**: só visível dentro do próprio pacote onde foi declarado; código de fora nem consegue enxergar que aquilo existe.

```go
package payment

func Process(amount float64) error { // Process: exportado, maiúscula — outros pacotes podem chamar
    return validateInternal(amount)
}

func validateInternal(amount float64) error { // validateInternal: não-exportado, minúscula
    if amount <= 0 {
        return errors.New("valor inválido")
    }
    return nil
}
```

Quem importa o pacote `payment` consegue chamar `payment.Process(...)`, mas não tem nenhuma forma de chamar `payment.validateInternal(...)` de fora — o compilador rejeita, porque essa função nem é visível fora do próprio pacote. Isso vale igual para campos de struct:

```go
type Order struct {
    ID     string  // exportado — visível de fora do pacote
    total  float64 // não-exportado — só código dentro deste mesmo pacote enxerga este campo
}
```

Essa regra é aplicada pelo próprio compilador, não é uma convenção que depende de disciplina de quem escreve o código — não tem como "furar" essa visibilidade de fora do pacote.

### Pacote pequeno e focado é Single Responsibility em nível de organização

Um pacote chamado `payment`, que só lida com processamento de pagamento, tem um único motivo claro pra mudar. Um pacote genérico chamado `utils` (comum em bases de código malorganizadas, de qualquer linguagem), que acumula funções desconexas sem relação entre si, mistura vários motivos de mudança num único lugar — é o mesmo problema de design que o **Single Responsibility Principle** descreve no nível de uma função ou de um tipo, só que aplicado ao nível de organização de pastas do projeto inteiro (definição completa em [contexts/common/SOLID.md](../../contexts/common/SOLID.md#s--single-responsibility-principle)). Nomear pacotes por **o que eles fazem no domínio do negócio** (`payment`, `order`, `catalog`) em vez de por **tipo técnico de conteúdo** (`utils`, `helpers`, `common`) é uma das formas mais simples e eficazes de manter essa responsabilidade única visível já na estrutura de pastas do projeto.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise tratamento de erros e pacotes`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
