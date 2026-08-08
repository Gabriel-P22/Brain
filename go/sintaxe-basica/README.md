# Sintaxe básica

## Setup: iniciando um módulo

Antes de escrever código Go organizado em um projeto, você precisa de um arquivo `go.mod` — ele declara o nome do módulo (geralmente o caminho onde o código vive, tipo `github.com/seu-usuario/seu-projeto`) e, conforme o projeto cresce, as dependências externas que ele usa. Para criar esse arquivo:

```
go mod init nome-do-modulo
```

Isso gera um `go.mod` na pasta atual. O tópico [Gerenciamento de pacotes](../gerenciamento-de-pacotes/) explica esse arquivo (e o `go.sum` que aparece junto quando há dependências) em detalhe.

Um único arquivo `.go` roda direto, sem precisar de módulo, com:

```
go run arquivo.go
```

Não existe um modo interativo padrão no Go — nenhum terminal onde você digita uma linha por vez e vê o resultado imediatamente. Todo código Go vive em um arquivo e roda a partir dele.

## Variáveis

Go é **estaticamente tipado** (o tipo de cada variável é fixo e checado antes do programa rodar — ver [O que é Go](../o-que-e-go/)), mas tem **inferência de tipo**: você não precisa escrever o tipo toda vez, o compilador consegue descobrir sozinho a partir do valor.

```go
var x int = 10   // forma explícita: tipo escrito na mão
var y = 10       // forma com inferência: o compilador deduz que y é int, pelo valor 10
z := 10          // forma curta: mesma coisa que "var z = 10", mas mais usada no dia a dia

const Pi = 3.14  // const é uma constante — resolvida em tempo de compilação, nunca muda depois
```

Detalhes importantes sobre cada forma:

- `var x int = 10` — a forma mais explícita. Útil quando você quer deixar o tipo bem claro, ou quando está declarando uma variável sem valor inicial ainda (ver "zero value" abaixo).
- `y := 10` — o operador `:=` declara **e** inicializa a variável na mesma linha, com o tipo inferido do valor à direita. É a forma mais comum dentro de funções. Importante: `:=` só funciona **dentro de uma função** — no nível mais externo de um arquivo (fora de qualquer função), você é obrigado a usar `var`.
- `const Pi = 3.14` — uma constante. Diferente de uma variável comum, o valor de uma `const` é resolvido durante a compilação e nunca pode ser reatribuído depois.

Depois que uma variável é declarada com um tipo, esse tipo é fixo para sempre — não existe reatribuir um valor de outro tipo:

```go
x := 10
x = "texto" // ERRO DE COMPILAÇÃO: cannot use "texto" (type string) as type int
```

## Tipos básicos

Cada variável em Go tem um tipo obrigatório, sempre. Não existe uma variável "sem tipo definido" ou que aceite qualquer coisa por padrão — isso é o que permite ao compilador pegar erros de tipo antes do programa rodar.

```go
var i int        // número inteiro; tamanho depende da plataforma (normalmente 64 bits) — use int como padrão
var i8 int8       // inteiro pequeno: de -128 a 127
var u uint        // inteiro sem sinal — só valores positivos (e zero)
var f float64      // número de ponto flutuante (com casas decimais); float64 é o padrão idiomático
var s string       // texto, sempre imutável (uma vez criada, uma string não pode ser alterada em memória)
var b bool          // true ou false — só isso, sem "valores que contam como falso" por coincidência
var by byte          // um byte de dado bruto; é só um outro nome (alias) para uint8
var r rune            // um "caractere" no sentido Unicode; é só um outro nome (alias) para int32
```

Por que existem tantos tamanhos de inteiro (`int8` até `int64`, `uint8` até `uint64`) em vez de um único tipo "número inteiro" genérico? Porque em código de infraestrutura e sistemas, o tamanho exato de um dado às vezes importa de verdade — para performance, para uso de memória, ou para bater exatamente com um formato binário externo (um protocolo de rede, por exemplo). Go deixa essa escolha explícita, em vez de esconder essa decisão. No dia a dia, a recomendação idiomática é simples: use `int` para inteiros e `float64` para decimais, a não ser que você tenha um motivo concreto para outro tamanho.

Sobre `bool`: em Go, `true` e `false` são os únicos dois valores possíveis de um `bool`. Não existe a ideia de "outros valores que também contam como falso" (como um número zero, ou um texto vazio, sendo tratados automaticamente como falso dentro de um `if`) — se você quer checar se um número é zero, escreve `if numero == 0`, explicitamente.

## Zero value: toda variável já nasce com um valor utilizável

Quando você declara uma variável sem dar um valor inicial, ela não fica "vazia" ou "indefinida" — Go automaticamente atribui o que se chama de **zero value**: um valor padrão específico para cada tipo.

```go
var i int       // 0
var s string     // "" (texto vazio, não algo tipo "sem valor")
var b bool        // false
var f float64      // 0.0
var p *int         // nil — "nenhum endereço de memória apontado" (só ponteiro, slice, map, interface, channel e função podem ser nil)
```

Isso é uma decisão de design importante: em Go, "ainda não inicializado" já é, por padrão, um valor pronto pra uso (0, texto vazio, falso) — na maioria dos casos você pode simplesmente começar a usar a variável sem precisar checar antes se ela "tem algum valor real". `nil` existe só para os tipos que representam "referência para algo que pode não existir" (ponteiro, slice, map, interface, channel, função) — é o único caso em Go parecido com a ideia de "nenhum valor aqui".

## Conversão de tipo: sempre explícita

Go nunca converte um tipo numérico para outro sozinho, mesmo quando pareceria óbvio fazer isso. Você sempre precisa converter manualmente:

```go
var i int = 10
var f float64 = float64(i)  // conversão explícita: float64(i) transforma o int i num float64
// f := i seria erro de compilação: mismatched types
```

Isso também vale em contas: dividir dois `int` produz um resultado `int` (a parte decimal é descartada, não arredondada), mesmo que matematicamente o resultado devesse ter casas decimais:

```go
resultado := 1 / 2      // resultado vale 0 — divisão inteira, a parte decimal simplesmente some
resultadoCerto := 1.0 / 2.0  // resultadoCerto vale 0.5 — porque os dois operandos já são float64
```

Se você tenta misturar um `int` com um `float64` diretamente numa operação, sem converter, o código nem compila:

```go
var i int = 1
var f float64 = 2.0
soma := i + f // ERRO DE COMPILAÇÃO: mismatched types int and float64
somaCerta := float64(i) + f // 3.0 — funciona, porque agora os dois lados são float64
```

## Funções são um tipo de valor

Em Go, uma função pode ser tratada como qualquer outro valor: guardada numa variável, passada como argumento para outra função, e até ter seu próprio tipo nomeado:

```go
type Validador func(string) error   // declara um tipo chamado Validador: qualquer função que recebe uma string e devolve um error

var v Validador = func(s string) error {
    if s == "" {
        return errors.New("texto vazio")
    }
    return nil
}

func rodar(v Validador, entrada string) error {
    return v(entrada)
}
```

Isso é bem mais do que um detalhe de sintaxe — é a base de como Go resolve, de forma simples, coisas que em outras abordagens exigiriam criar uma classe inteira só para representar "um comportamento configurável". Aqui, `Validador` é só uma variável do tipo certo, que pode ser trocada por outra implementação sem precisar de herança nem hierarquia nenhuma.

## Controle de fluxo

Go tem apenas uma palavra-chave de repetição: `for`. Não existe `while`, nem `do-while`, nem nenhum outro tipo de laço — `for` cobre todos os casos, com formas diferentes de escrever a condição:

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}    // a forma clássica: inicialização, condição, incremento

contador := 0
for contador < 5 {
    contador++
}    // só a condição — funciona como o "enquanto" de outras linguagens

for {
    // corpo do laço
    break // sem essa linha (ou algum outro jeito de sair), esse laço nunca termina sozinho
}    // loop infinito, controlado manualmente com break
```

O `if` em Go não usa parênteses ao redor da condição, e aceita uma instrução de inicialização antes da condição, separada por ponto e vírgula:

```go
if err := fazerAlgo(); err != nil {
    return err
}
// a variável err só existe dentro deste bloco if/else — fora dele, ela não existe mais
```

Essa forma (`if <inicialização>; <condição> { }`) é usada o tempo todo em Go, especialmente com tratamento de erro (assunto do tópico [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)). A vantagem prática: a variável `err` fica restrita ao menor escopo possível — ela não "vaza" para o resto da função, o que deixa o código mais fácil de acompanhar, porque cada variável só existe exatamente onde é necessária.

## Funções com múltiplos retornos

Diferente de muitas linguagens, onde uma função só devolve um único valor, funções em Go podem devolver mais de um valor ao mesmo tempo — e isso é usado o tempo inteiro para retornar "o resultado" junto com "o erro, se algo deu errado":

```go
func dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divisão por zero")
    }
    return a / b, nil
}

resultado, err := dividir(10, 2)
if err != nil {
    fmt.Println("erro:", err)
    return
}
fmt.Println("resultado:", resultado) // 5
```

Repare que quem chama `dividir` recebe dois valores de volta (`resultado` e `err`) e é obrigado a lidar com os dois — não existe forma de "pular" o segundo valor de retorno sem escrever algo explícito para isso. Se você realmente quiser ignorar um valor de retorno, precisa fazer isso de forma visível, usando `_` (chamado de "identificador em branco"):

```go
resultado, _ := dividir(10, 2) // ignora o erro de propósito — visível pra quem ler o código depois
```

Esse padrão — `resultado, err := algumaFuncao()` seguido de `if err != nil { ... }` — é provavelmente a estrutura de código mais comum que você vai ver em qualquer programa Go real.

## Exercício

Sem exercício gerado por padrão neste tópico por enquanto — quando quiser praticar, rode `/exercise sintaxe básica`; o código vai em `exercise/` (fora do git, ver `.gitignore`).

`exercise/main.go` já tem um rascunho: soma de 1 a 5 num `for`, `divide` com múltiplo retorno tratando divisão por zero via `if` com inicialização escopada.
