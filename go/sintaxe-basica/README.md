# Sintaxe básica

## Setup

`go mod init <nome>` é o equivalente do `pyproject.toml`/`requirements.txt` — declara o módulo e (futuramente) as dependências. Um arquivo `.go` roda direto com `go run arquivo.go`, sem REPL padrão como o `python`.

## Variáveis

Go é estaticamente tipado, mas com inferência:

```go
var x int = 10   // explícito
y := 10           // inferido — := só funciona dentro de função, não em nível de pacote
const Pi = 3.14    // const é resolvido em compile-time, diferente do "const por convenção" do Python
```

Sem `Any` implícito: toda variável tem tipo fixo depois de declarada. Não existe reatribuir `x = "texto"` depois de `x := 10` — o compilador rejeita.

## Tipos

Em Python o sistema de tipos é dinâmico (o valor carrega o tipo, a variável não) e "opcional" (type hints são checados por ferramenta externa, não pelo interpretador). Em Go o tipo é da variável, checado em compile-time, e sempre obrigatório — isso é o que torna Go rápido e permite pegar erro de tipo antes de rodar, não em produção.

### Tipos básicos

```go
var i int        // tamanho da plataforma (64 bits normalmente) — use int por padrão
var i8 int8       // -128 a 127
var u uint        // sem sinal — só positivo
var f float64      // padrão pra ponto flutuante (float32 existe, mas float64 é o default idiomático)
var s string       // imutável, como str em Python — mas iterar sobre string itera bytes/runes, não "caracteres" direto
var b bool          // true/false, sem truthiness de valor (0, "", nil NÃO viram false implicitamente como em Python)
var by byte          // alias de uint8 — usado pra dado bruto/binário
var r rune            // alias de int32 — um "caractere" Unicode (equivalente mais próximo do str[i] de um Python que lida com Unicode)
```

Diferente de Python (só `int`, sem limite de tamanho, e `float` = double sempre), em Go você escolhe explicitamente a largura (`int8`...`int64`, `uint8`...`uint64`) quando importa — isso importa em código de infra/performance, onde tamanho de dado é decisão consciente, não incidental.

### Zero value — não existe `None`/`null` "solto"

Toda variável declarada sem valor inicial recebe o **zero value** do seu tipo, nunca um estado "vazio" universal:

```go
var i int       // 0
var s string     // "" (string vazia, não nil)
var b bool        // false
var p *int         // nil (só ponteiro, slice, map, interface, channel, func têm nil)
```

Isso substitui boa parte do uso de `None` do Python como "ainda não inicializado" — em Go, "ainda não inicializado" já é um valor utilizável (0, "", false), não precisa checar `is None` antes de usar na maioria dos casos.

### Conversão explícita — sem coerção implícita

```go
var i int = 10
var f float64 = float64(i)  // conversão explícita obrigatória
// f := i seria erro de compilação: mismatched types
```

Python converte `int` pra `float` automaticamente numa divisão (`1 / 2 == 0.5`); Go não — `1 / 2` entre `int` é `0` (divisão inteira), e misturar tipo numérico diferente sem conversão explícita nem compila.

### Funções são tipo de primeira classe

```go
type Validator func(string) error   // função é um tipo, pode ter nome próprio

var v Validator = func(s string) error {
    if s == "" {
        return errors.New("vazio")
    }
    return nil
}

func run(v Validator, input string) error { return v(input) }
```

Em Python isso é natural (função é objeto). Em Go é mais explícito: o tipo da função (parâmetros + retorno) é a assinatura, e isso é a base de como interface pequena + função batem — muita coisa que em outras linguagens vira classe com um método vira, em Go, simplesmente uma variável do tipo `func(...) ...`.

## Controle de fluxo

Go só tem `for` — não existe `while`, nem list comprehension:

```go
for i := 0; i < 10; i++ { }   // clássico
for condition { }              // vira o "while"
for { }                        // loop infinito, break manual
```

`if` não usa parênteses e aceita uma inicialização escopada:

```go
if err := doSomething(); err != nil {
    return err
}
// err só existe dentro desse if/else — não vaza pro resto da função
```

Isso já é o embrião do padrão de erro do Dia 4 (erro é valor, checado explicitamente, não exceção) e uma aplicação implícita de escopo mínimo — mesmo espírito do Single Responsibility ([contexts/common/SOLID.md](../../contexts/common/SOLID.md)): cada coisa só existe onde precisa existir.

## Funções e múltiplo retorno

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divisão por zero")
    }
    return a / b, nil
}
```

Não existe exception aqui — quem chama é obrigado a olhar os dois valores. Em Python, `divide` levantaria `ZeroDivisionError` e o chamador podia simplesmente não tratar; em Go, ignorar o erro é uma escolha visível (`_, _ = divide(...)`), não um esquecimento silencioso.

## Exercício

Sem exercício gerado por padrão neste tópico por enquanto — quando quiser praticar, rode `/exercise sintaxe básica`; o código vai em `exercise/` (fora do git, ver `.gitignore`).

`exercise/main.go` já tem um rascunho: soma de 1 a 5 num `for`, `divide` com múltiplo retorno tratando divisão por zero via `if` com inicialização escopada.
