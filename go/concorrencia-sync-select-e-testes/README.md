# Concorrência — sync, select e testes

## WaitGroup — esperando um número conhecido de goroutines terminarem

No tópico anterior ([Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/)), você viu que `go` dispara uma goroutine e segue em frente sem esperar ela terminar — e que, para fins didáticos, usamos `time.Sleep` para dar tempo dela rodar. Isso não é uma solução real: `time.Sleep` é uma aposta ("provavelmente um segundo é tempo suficiente"), não uma garantia. A ferramenta correta e idiomática, dentro do pacote `sync` da biblioteca padrão, para esperar de verdade um grupo de goroutines terminarem, é o `sync.WaitGroup`:

```go
package main

import (
    "fmt"
    "sync"
)

func fazerTrabalho(id int) {
    fmt.Println("goroutine", id, "trabalhando")
}

func main() {
    var wg sync.WaitGroup // zero value já é utilizável, sem precisar de construtor

    for i := 0; i < 3; i++ {
        wg.Add(1) // avisa o WaitGroup: "existe mais uma goroutine pendente para esperar"
        go func(id int) {
            defer wg.Done() // avisa o WaitGroup: "esta goroutine terminou"
            fazerTrabalho(id)
        }(i)
    }

    wg.Wait() // bloqueia aqui até TODAS as chamadas de Add terem seu Done() correspondente
    fmt.Println("todas as goroutines terminaram")
}
```

O funcionamento do `WaitGroup` é um contador simples por baixo dos panos:

- `wg.Add(1)` soma 1 ao contador interno — chamado antes de disparar cada goroutine que você quer esperar.
- `wg.Done()` subtrai 1 do contador — chamado de dentro de cada goroutine, no momento em que ela termina o trabalho dela.
- `wg.Wait()` bloqueia a execução de quem chamou, até o contador voltar a zero.

Repare em `defer wg.Done()`: `defer` é uma instrução de Go que agenda uma chamada de função para rodar **quando a função atual terminar de executar**, não importa como ela termine — mesmo que termine por um `return` no meio do código, ou até por um `panic`. Isso é importante aqui porque garante que `Done()` sempre vai ser chamado, mesmo que `fazerTrabalho` dê algum tipo de erro inesperado no meio — sem esse `defer`, um erro no meio do caminho poderia deixar o `WaitGroup` esperando para sempre por uma goroutine que nunca vai chamar `Done()`.

Note também o parâmetro `id int` na função anônima, recebido como argumento (`func(id int) { ... }(i)`) em vez de simplesmente usar `i` direto de dentro da closure. Isso não é mais estritamente necessário para correção a partir do Go 1.22 (como visto no tópico anterior, cada iteração do `for` já tem sua própria cópia de `i`), mas ainda é uma forma clara e explícita de indicar que aquele valor pertence àquela goroutine específica — muita gente mantém esse estilo por clareza, mesmo sem ser mais obrigatório para evitar bug.

## Mutex — quando duas goroutines precisam mexer no mesmo dado

Channel resolve o problema de **comunicação** entre goroutines (mandar um valor de uma para outra). Mas existe um problema diferente: e se duas ou mais goroutines precisam **ler e escrever a mesma variável** ao mesmo tempo? Isso é chamado de **condição de corrida** (race condition) — quando o resultado final do programa depende da ordem exata (imprevisível) em que as operações de cada goroutine acontecem, o que costuma produzir resultados errados e difíceis de reproduzir.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var contador int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            contador++ // PERIGO: mil goroutines tentando incrementar a mesma variável ao mesmo tempo
        }()
    }

    wg.Wait()
    fmt.Println(contador) // resultado imprevisível — normalmente MENOR que 1000
}
```

O motivo desse resultado imprevisível: `contador++` não é uma operação única e instantânea — por baixo dos panos, ela envolve ler o valor atual de `contador`, somar 1, e gravar o resultado de volta. Se duas goroutines fazem essas três etapas ao mesmo tempo, é possível que ambas leiam o mesmo valor antes de qualquer uma gravar o resultado — e aí um dos dois incrementos "se perde".

A ferramenta para resolver isso é `sync.Mutex` ("mutual exclusion" — exclusão mútua): um mecanismo de trava que garante que só uma goroutine por vez consiga executar um trecho específico de código.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var mu sync.Mutex
    var contador int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()         // pega a trava — se outra goroutine já está com ela, espera aqui
            defer mu.Unlock() // libera a trava quando esta função terminar
            contador++         // agora só uma goroutine por vez executa esta linha
        }()
    }

    wg.Wait()
    fmt.Println(contador) // sempre 1000, de forma confiável
}
```

`mu.Lock()` pega a trava — se outra goroutine já a tiver pego antes e ainda não a liberou, esta goroutine fica bloqueada esperando até a trava ficar livre. `mu.Unlock()` libera a trava para a próxima goroutine que estiver esperando. O padrão `mu.Lock()` seguido imediatamente de `defer mu.Unlock()` é o jeito idiomático de garantir que a trava sempre vai ser liberada, mesmo que algo dê errado no meio do trecho protegido.

### Detectando condições de corrida automaticamente

Go tem uma ferramenta embutida na própria toolchain para detectar esse tipo de bug: a flag `-race`.

```
go run -race main.go
go test -race ./...
```

Rodar o programa (ou os testes) com `-race` faz o Go monitorar, durante a execução, todos os acessos concorrentes a variáveis compartilhadas, e avisa explicitamente no terminal se encontrar um acesso não protegido — mesmo que, por sorte, o programa tenha rodado sem produzir um resultado visivelmente errado daquela vez (condições de corrida são notoriamente inconsistentes: podem passar despercebidas em várias execuções e só se manifestar ocasionalmente). É uma prática recomendada rodar os testes com `-race` sempre que o código envolve concorrência.

## select — esperando em vários channels ao mesmo tempo

`select` funciona como uma bifurcação (parecido com um `switch`, mas voltado especificamente para operações de channel): ele espera em várias operações de channel simultaneamente, e segue com o primeiro `case` que ficar pronto:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "resultado do ch1"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "resultado do ch2"
    }()

    for i := 0; i < 2; i++ {
        select {
        case v := <-ch1:
            fmt.Println("veio de ch1:", v)
        case v := <-ch2:
            fmt.Println("veio de ch2:", v)
        }
    }
}
```

Nesse exemplo, `ch1` recebe um valor depois de 1 segundo, e `ch2` depois de 2 segundos. O `select`, em cada volta do `for`, espera até que algum dos dois channels tenha um valor pronto para receber, e segue por aquele `case` — na prática, isso imprime primeiro "veio de ch1" e, na segunda volta, "veio de ch2".

Um uso muito comum de `select` é implementar um timeout — um limite de tempo depois do qual o programa desiste de esperar por um resultado:

```go
select {
case v := <-ch1:
    fmt.Println("recebido a tempo:", v)
case <-time.After(2 * time.Second):
    fmt.Println("timeout: esperou 2 segundos e nada chegou")
}
```

`time.After(2 * time.Second)` devolve um channel que, sozinho, "dispara" (envia um valor) depois do tempo indicado ter passado. Colocado como um `case` dentro do `select`, ele funciona como um limite de espera: se `ch1` não enviar nada dentro desses 2 segundos, o `case` do `time.After` é o que acaba sendo escolhido. Não existe um mecanismo separado e especial de "timeout" na linguagem — um timeout, em Go, é só mais um channel, tratado exatamente como qualquer outro dentro de um `select`.

`select` também aceita um `default`, que roda imediatamente se **nenhum** dos outros `case` estiver pronto naquele instante (em vez de bloquear esperando um deles ficar pronto):

```go
select {
case v := <-ch1:
    fmt.Println("tinha algo pronto:", v)
default:
    fmt.Println("nada pronto agora, seguindo sem esperar")
}
```

## Testes em Go

Go tem um framework de testes embutido na própria biblioteca padrão, no pacote `testing` — sem precisar instalar nenhuma ferramenta externa para escrever e rodar testes automatizados. A convenção é: um arquivo de teste tem o mesmo nome do arquivo que ele testa, com o sufixo `_test.go` (por exemplo, `divide.go` e `divide_test.go`), e cada função de teste começa com `Test`, recebe um parâmetro `*testing.T`, e não devolve nada:

```go
// arquivo: divide.go
package math

import "errors"

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("divisão por zero")
    }
    return a / b, nil
}
```

```go
// arquivo: divide_test.go
package math

import "testing"

func TestDivide(t *testing.T) {
    resultado, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("não esperava erro, recebi: %v", err)
    }
    if resultado != 5 {
        t.Errorf("esperava 5, recebi %v", resultado)
    }
}
```

Para rodar todos os testes do projeto:

```
go test ./...
```

A diferença entre `t.Fatalf` e `t.Errorf`: `t.Errorf` marca o teste como falho e **continua** executando o resto da função de teste (útil quando você quer ver todas as falhas de uma vez, não só a primeira); `t.Fatalf` marca o teste como falho e **interrompe** a execução dessa função de teste imediatamente (útil quando não faz sentido continuar checando o resto, porque uma etapa anterior já falhou de um jeito que invalida o resto do teste).

### O padrão idiomático: table-driven tests

Testar vários cenários diferentes de uma mesma função, um por um, com uma função de teste separada para cada caso, gera muita repetição. O padrão idiomático em Go para isso é **table-driven test**: uma "tabela" (uma slice de structs) descrevendo cada caso, percorrida por um único laço, com cada caso rodando como um subteste nomeado através de `t.Run`:

```go
func TestDivide(t *testing.T) {
    casos := []struct {
        nome    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {nome: "divisão normal", a: 10, b: 2, want: 5, wantErr: false},
        {nome: "divisão por zero", a: 10, b: 0, want: 0, wantErr: true},
        {nome: "resultado negativo", a: -10, b: 2, want: -5, wantErr: false},
    }

    for _, c := range casos {
        t.Run(c.nome, func(t *testing.T) {
            got, err := Divide(c.a, c.b)

            if (err != nil) != c.wantErr {
                t.Fatalf("erro = %v, wantErr = %v", err, c.wantErr)
            }
            if !c.wantErr && got != c.want {
                t.Errorf("got = %v, want = %v", got, c.want)
            }
        })
    }
}
```

Vantagens desse padrão: adicionar um novo cenário de teste vira só uma linha nova na tabela `casos` (sem duplicar a lógica de checagem), e `t.Run(c.nome, ...)` nomeia cada subteste individualmente na saída do `go test` — o que facilita muito identificar exatamente qual cenário falhou quando algo dá errado:

```
--- FAIL: TestDivide/divisão_por_zero
    divide_test.go:23: erro = <nil>, wantErr = true
```

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise concorrência sync select e testes`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
