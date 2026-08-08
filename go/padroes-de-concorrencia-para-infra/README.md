# Padrões de concorrência para infra

Aplicação prática do que já foi explicado do zero em [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) (o que é uma goroutine, o que é um channel) e [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/) (`sync.WaitGroup`, `select`) — nenhum conceito novo de linguagem aqui, só composição do que já existe pra resolver três problemas concretos e recorrentes em código de infraestrutura.

## O problema: goroutine sem limite pode esgotar recurso

Disparar uma goroutine por item de uma lista, sem nenhum controle, parece simples:

```go
// versão ingênua: uma goroutine por item, sem limite nenhum
func processAll(items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        go func() {
            defer wg.Done()
            process(item) // se process() abrir uma conexão de banco, e items tiver 50 mil itens...
        }()
    }
    wg.Wait()
}
```

O problema aparece quando `items` tem uma quantidade grande de elementos e `process` faz algo caro — abrir uma conexão de banco, chamar uma API externa. Disparar 50 mil goroutines de uma vez pode esgotar o pool de conexões do banco, estourar limite de conexão simultânea de uma API externa, ou simplesmente consumir memória demais. Uma goroutine é barata (ver [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/)), mas "barata" não é "grátis" — cada uma ainda ocupa memória e, mais importante, cada uma pode estar segurando um recurso externo caro (uma conexão) ao mesmo tempo que as outras.

## Worker pool — limitar concorrência a N workers fixos

A correção estrutural é fixar um número de "trabalhadores" (goroutines) que ficam vivos o tempo todo, consumindo trabalho de um channel — nunca mais que N goroutines rodando `process` ao mesmo tempo, não importa quantos itens existam:

```go
func processAll(items []Item, workers int) {
    jobs := make(chan Item) // channel de trabalho — os workers consomem daqui
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs { // cada worker fica pegando itens até o channel fechar
                process(item)
            }
        }()
    }

    for _, item := range items {
        jobs <- item // envia cada item pro channel — bloqueia se todos os workers estiverem ocupados
    }
    close(jobs)   // sinaliza "não vem mais trabalho" — workers saem do for/range e chamam Done()
    wg.Wait()     // espera todos os workers realmente terminarem o que já pegaram
}
```

O limite aqui é estrutural, não um número decorativo: só existem `workers` goroutines rodando `process` ao mesmo tempo, porque só existem `workers` goroutines no total, todas competindo pra ler do mesmo channel `jobs`. Quando um worker termina um item, ele volta pro topo do `for range jobs` e pega o próximo disponível — é um balanceamento de carga automático, sem nenhuma lógica extra pra distribuir trabalho entre workers.

## Coletando resultados de volta

O worker pool acima processa cada item, mas descarta qualquer resultado (`process` não devolve nada usado). Uma variação comum precisa capturar o resultado de cada item processado — nesse caso, entra um segundo channel, de saída:

```go
type Result struct {
    Item  Item
    Err   error
}

func processAllWithResults(items []Item, workers int) []Result {
    jobs := make(chan Item)
    results := make(chan Result)
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                err := process(item)
                results <- Result{Item: item, Err: err}
            }
        }()
    }

    go func() {
        for _, item := range items {
            jobs <- item
        }
        close(jobs)
    }()

    go func() {
        wg.Wait()       // espera todos os workers acabarem...
        close(results)  // ...só então fecha o channel de resultados
    }()

    var out []Result
    for r := range results { // consome os resultados conforme chegam
        out = append(out, r)
    }
    return out
}
```

Duas goroutines auxiliares aparecem aqui: uma só pra alimentar `jobs` (sem ela, o envio de itens bloquearia a função inteira até o primeiro worker consumir), e outra só pra fechar `results` assim que todos os workers terminarem — fechar esse channel cedo demais faria a última leitura de `for r := range results` travar pra sempre (goroutine leak, ver [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/#duas-armadilhas-clássicas)).

## Rate limiting — controlar a taxa, não só a quantidade simultânea

Worker pool limita "quantos ao mesmo tempo". Rate limiting resolve um problema diferente: "no máximo quantos por segundo", mesmo que só exista um worker. Isso importa quando uma API externa tem um limite de requisições por segundo, e passar disso resulta em erro `429 Too Many Requests` — ou pior, banimento temporário.

```go
limiter := time.NewTicker(100 * time.Millisecond) // um "tick" a cada 100ms = no máximo 10 por segundo
defer limiter.Stop()

for _, req := range requests {
    <-limiter.C // bloqueia até o próximo tick chegar
    doRequest(req)
}
```

`time.NewTicker` cria um relógio que envia um valor pro channel `limiter.C` em intervalos regulares. `<-limiter.C` bloqueia a goroutine atual até o próximo tick — na prática, isso espaça as chamadas de `doRequest` no ritmo definido, mesmo que o resto do código consiga rodar bem mais rápido que isso. Pra caso de uso real (não só didático), o pacote oficial `golang.org/x/time/rate` (biblioteca extra, mantida pelo time do Go mas fora da stdlib) implementa um token bucket completo, com suporte a rajada (`burst`) — o exemplo com `time.Ticker` acima é a versão mínima do mecanismo, útil pra entender o princípio antes de usar a versão pronta.

## Pipeline — estágios encadeados por channel

Um terceiro padrão comum: dividir um processamento em etapas sequenciais, cada uma rodando na sua própria goroutine, conectadas por channels — o dado flui de uma etapa pra outra, sendo processado concorrentemente estágio a estágio:

```go
func generate(nums []int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}

// uso: encadeia os três estágios — generate produz, square transforma, filterEven filtra
for v := range filterEven(square(generate([]int{1, 2, 3, 4, 5}))) {
    fmt.Println(v) // 4, 16 (quadrados pares de 1..5)
}
```

Cada função devolve um `<-chan int` (channel só-de-leitura, do ponto de vista de quem recebe — a seta indica a direção permitida) e roda sua própria goroutine internamente, fechando o channel de saída (`defer close(out)`) quando termina de processar tudo que recebeu. `generate` não recebe de ninguém (é a fonte), `square` e `filterEven` recebem de um estágio anterior e produzem pro próximo. Nenhuma goroutine central está coordenando esse fluxo — cada estágio sabe só ler do seu `in` e escrever no seu `out`, e o encadeamento inteiro (`filterEven(square(generate(...)))`) é só composição de funções comuns.
