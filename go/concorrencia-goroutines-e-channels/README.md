# Concorrência — goroutines e channels

## O que "concorrência" significa

Até agora, todo código que você viu roda de um jeito **sequencial**: uma instrução termina, só então a próxima começa, uma de cada vez, na ordem em que estão escritas no arquivo. Isso funciona bem para a maioria dos programas simples, mas tem um problema real em situações como: um servidor que precisa atender centenas de pedidos de clientes diferentes ao mesmo tempo, ou um programa que precisa baixar 100 arquivos da internet — se cada download precisasse esperar o anterior terminar completamente antes de começar o próximo, o programa inteiro ficaria muito mais lento do que precisaria ser, já que a maior parte do tempo de um download é gasto esperando a rede responder, não processando de verdade.

**Concorrência** é a capacidade de um programa ter várias tarefas "em andamento" ao mesmo tempo, sem que uma tarefa lenta trave todas as outras. Isso é considerado o maior diferencial de Go como linguagem (mencionado já em [O que é Go](../o-que-e-go/#concorrência-nativa--o-maior-diferencial-de-go)): a linguagem tem sintaxe própria, embutida na própria gramática, especificamente para isso — não é uma biblioteca externa que você precisa instalar, é parte do núcleo da linguagem.

## Goroutine — uma unidade de execução concorrente, leve e gerenciada pelo runtime

A forma de disparar uma tarefa para rodar de forma concorrente com o resto do programa é colocar a palavra-chave `go` na frente de uma chamada de função:

```go
package main

import (
    "fmt"
    "time"
)

func dizerOla() {
    fmt.Println("olá do meio de uma goroutine")
}

func main() {
    go dizerOla() // dispara e NÃO espera terminar — a próxima linha já roda em seguida

    fmt.Println("olá do main")
    time.Sleep(100 * time.Millisecond) // sem isso, o programa pode terminar antes da goroutine rodar
}
```

Essa unidade de execução que o `go` dispara se chama **goroutine**. Ela é gerenciada pelo próprio runtime de Go (o programa que fica "por trás" rodando junto com o seu código, cuidando de coisas como isso), e não diretamente pelo sistema operacional — isso a torna muito mais barata, em termos de memória e de tempo para criar e trocar de contexto, do que as unidades de execução concorrente tradicionais gerenciadas pelo sistema operacional. Um programa Go comum consegue ter milhares, até milhões, de goroutines rodando ao mesmo tempo sem problema de desempenho.

Um ponto importante sobre o exemplo acima: `go dizerOla()` dispara a goroutine e **imediatamente** segue para a próxima linha, sem esperar `dizerOla` terminar. Se o programa (a função `main`) terminar antes da goroutine ter chance de rodar, ela é simplesmente abandonada — Go não espera goroutines pendentes terminarem sozinho quando `main` acaba. É por isso que o exemplo usa `time.Sleep` ali: só para dar tempo da goroutine rodar antes do programa encerrar. Isso é só para fins didáticos — a forma correta e idiomática de esperar uma ou mais goroutines terminarem de verdade é usando `sync.WaitGroup`, assunto do próximo tópico, [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/#waitgroup--esperar-n-goroutines-terminarem).

Uma goroutine também pode ser uma função anônima (uma função sem nome, declarada ali mesmo, na hora de usar):

```go
go func() {
    fmt.Println("goroutine anônima")
}()
```

## Channel — o jeito idiomático de goroutines se comunicarem

Assim que você tem várias goroutines rodando ao mesmo tempo, surge uma pergunta inevitável: como uma goroutine manda um resultado para outra parte do programa, de forma segura, sem duas goroutines lerem e escreverem no mesmo dado ao mesmo tempo de um jeito bagunçado (o que se chama de "condição de corrida" — assunto que volta com mais força no próximo tópico)? Go resolve isso de um jeito específico, resumido numa frase comum na comunidade da linguagem: "compartilhe memória comunicando, em vez de comunicar compartilhando memória". Ou seja: em vez de duas goroutines acessarem a mesma variável diretamente (o que exigiria travas manuais para não bagunçar tudo), elas trocam dados através de um canal de comunicação dedicado, chamado **channel**.

```go
package main

import "fmt"

func main() {
    ch := make(chan int) // cria um channel de int, não-bufferizado

    go func() {
        ch <- 42 // ENVIA o valor 42 para dentro do channel
    }()

    v := <-ch // RECEBE um valor do channel
    fmt.Println(v) // 42
}
```

`make(chan int)` cria um channel — pense nele como um "tubo" por onde valores de um tipo específico (`int`, nesse caso) podem passar de uma goroutine para outra. `ch <- 42` é a sintaxe para **enviar** um valor para dentro do channel; `<-ch` é a sintaxe para **receber** um valor de dentro dele.

Um channel criado sem nenhuma capacidade extra, como `make(chan int)` no exemplo acima, é chamado de **não-bufferizado**: o envio (`ch <- 42`) só é concluído no exato momento em que outra goroutine estiver, ao mesmo tempo, pronta para receber (`<-ch`). Se nenhuma goroutine estiver esperando para receber, o envio **bloqueia** — fica parado esperando — até que alguém receba. Da mesma forma, receber de um channel vazio bloqueia até que alguém envie algo. Isso, por si só, já funciona como um ponto de sincronização entre duas goroutines: as duas "se encontram" exatamente naquele momento.

### Channel bufferizado — desacopla envio e recebimento até um certo limite

Um channel pode ser criado com uma capacidade — um espaço de "buffer" onde valores enviados ficam esperando, mesmo que ninguém tenha recebido ainda:

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 3) // channel bufferizado, capacidade 3

    ch <- 1 // não bloqueia — ainda cabe no buffer
    ch <- 2 // não bloqueia
    ch <- 3 // não bloqueia — buffer cheio agora (3 de 3)

    fmt.Println(<-ch) // 1 — recebe na mesma ordem em que foi enviado
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
}
```

Enquanto o buffer não estiver cheio, enviar um valor não bloqueia (a goroutine que envia não precisa esperar ninguém receber naquele momento). Só quando o buffer enche é que um novo envio volta a bloquear, esperando que algum espaço se abra (alguém receber um valor de lá).

## Fechando um channel e lendo com range

Quando quem envia sabe que não vai mandar mais nenhum valor, é convenção fechar o channel com `close`, e quem consome pode usar `for ... range` para ler todos os valores até o channel fechar:

```go
package main

import "fmt"

func gerarNumeros(ch chan<- int) { // chan<- int: este parâmetro só pode ser usado para ENVIAR
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // sinaliza "não vem mais nada" — só quem envia deveria fechar o channel
}

func main() {
    ch := make(chan int)
    go gerarNumeros(ch)

    for v := range ch { // lê valores um a um, até o channel fechar
        fmt.Println(v)
    }
    fmt.Println("channel fechado, loop terminou sozinho")
}
```

Repare no tipo do parâmetro de `gerarNumeros`: `chan<- int` (em vez de só `chan int`) é uma forma de restringir esse channel, dentro dessa função, a só permitir envio — é uma forma de deixar explícito, na própria assinatura da função, qual é o papel esperado daquele channel ali (o oposto, `<-chan int`, restringiria a só recebimento). O compilador garante essa restrição: tentar receber de um `chan<- int` dentro dessa função nem compila.

Ler de um channel já fechado **não gera erro nem panic** — simplesmente devolve o zero value do tipo imediatamente, para sempre. Se você precisa diferenciar "recebi um zero value de verdade" de "o channel está fechado", existe a forma de dois retornos:

```go
v, ok := <-ch
if !ok {
    fmt.Println("channel está fechado, não vem mais nada")
} else {
    fmt.Println("recebido:", v)
}
```

`ok` é `false` quando o channel está fechado e não há mais valores pendentes no buffer para receber; é `true` em qualquer outro caso.

## Duas armadilhas clássicas de concorrência em Go

### Goroutine leak — uma goroutine que nunca termina

Se uma goroutine fica bloqueada esperando um channel que nunca vai receber ou enviar nada (por exemplo, porque a outra ponta daquela comunicação já terminou, ou nunca existiu de verdade), essa goroutine simplesmente **fica presa para sempre**. Isso não gera erro nenhum visível — o programa continua rodando normalmente, só que com uma goroutine a mais consumindo memória, indefinidamente, nunca coletada:

```go
func vazamento() {
    ch := make(chan int) // não-bufferizado, ninguém nunca vai receber
    go func() {
        ch <- 1 // bloqueia aqui PARA SEMPRE, porque ninguém nunca lê de ch
    }()
    // a função termina aqui, sem nunca ler de ch — a goroutine acima fica presa eternamente
}
```

Isso é chamado de "goroutine leak" (vazamento de goroutine). Não existe um aviso automático do compilador para isso — ferramentas de análise de performance da própria toolchain do Go (como `pprof`, abordado no Módulo 3) ajudam a detectar esse tipo de problema em um programa já rodando, mostrando quantas goroutines estão ativas e o que cada uma está esperando.

### Variável de loop capturada por uma closure

Uma **closure** é uma função anônima que "lembra" e consegue acessar variáveis do escopo onde ela foi criada, mesmo depois desse escopo ter terminado de executar. Isso costumava causar um bug muito comum quando combinado com `go` dentro de um `for`:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    items := []string{"a", "b", "c"}

    for _, v := range items {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(v)
        }()
    }
    wg.Wait()
}
```

Rodando esse exato código na versão atual da linguagem instalada neste ambiente (Go 1.26), o resultado é `a`, `b`, `c`, cada um impresso exatamente uma vez, em alguma ordem (a ordem entre goroutines diferentes não é garantida, mas os três valores aparecem certos). Isso porque, **a partir do Go 1.22**, cada repetição do `for` cria sua própria cópia independente da variável `v` — cada goroutine "vê" o valor de `v` daquela iteração específica em que foi criada, sem interferência das outras.

Vale saber que isso nem sempre foi assim: em versões anteriores ao Go 1.22, `v` era uma **única variável reaproveitada** a cada volta do `for`, e todas as goroutines disparadas dentro do loop acabavam enxergando o mesmo `v`, com o valor que ele tinha no momento em que cada goroutine finalmente rodasse — normalmente já o último valor da lista, repetido três vezes. Isso é considerado um dos bugs mais famosos e mais comuns da história da linguagem. Se você encontrar um tutorial ou um código mais antigo com um trecho estranho como este dentro do loop, é exatamente o workaround manual que existia antes da correção:

```go
for _, v := range items {
    v := v // cria uma cópia NOVA de v, própria daquela iteração — necessário só em Go < 1.22
    go func() {
        fmt.Println(v)
    }()
}
```

Como este ambiente já roda uma versão recente do Go, você não precisa mais escrever esse workaround — mas reconhecer esse padrão em código legado (ou em respostas antigas na internet) é útil.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise concorrência goroutines e channels`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
