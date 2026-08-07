# Padrões de concorrência para infra

Aplicação prática de [goroutines e channels](../concorrencia-goroutines-e-channels/) e [sync/select](../concorrencia-sync-select-e-testes/) em problema real de infra — não conceito novo, composição do que já foi visto.

## Worker pool — limitar concorrência

Disparar uma goroutine por item sem limite pode esgotar conexão de banco/rede. Worker pool fixa N workers consumindo de um channel de trabalho:

```go
func processAll(items []Item, workers int) {
    jobs := make(chan Item)
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {   // consome até o channel fechar
                process(item)
            }
        }()
    }

    for _, item := range items {
        jobs <- item
    }
    close(jobs)   // sinaliza "acabou" — workers saem do for/range e chamam Done()
    wg.Wait()
}
```

Comparável a `ThreadPoolExecutor`/`asyncio.Semaphore` do Python, mas o limite é estrutural (N goroutines fixas lendo de um channel), não um contador decorativo em cima de chamada assíncrona.

## Rate limiting — token bucket

```go
limiter := time.NewTicker(100 * time.Millisecond)   // 1 token a cada 100ms = 10 req/s
defer limiter.Stop()

for _, req := range requests {
    <-limiter.C   // bloqueia até o próximo tick
    doRequest(req)
}
```

Pra caso de uso real, `golang.org/x/time/rate` (pacote oficial extra) implementa token bucket completo com burst — o `time.Ticker` acima é a versão didática do mecanismo.

## Pipeline — estágios encadeados por channel

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

// uso: for v := range square(generate(nums)) { ... }
```

Cada estágio é uma goroutine própria conectada por channel — o dado flui em pipeline, processado concorrentemente estágio a estágio, sem uma goroutine central coordenando tudo. Padrão sem equivalente direto simples em Python (dá pra montar com `asyncio.Queue`, mas não é o idiom natural da linguagem como é em Go).
