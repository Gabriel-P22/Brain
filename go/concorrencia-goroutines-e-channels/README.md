# Concorrência — goroutines e channels

## Goroutine — thread leve, gerenciada pelo runtime

```go
go doSomething()   // roda concorrentemente, não bloqueia quem chamou
```

`go` na frente de uma chamada dispara uma goroutine — uma "thread" gerenciada pelo runtime do Go (não pelo SO diretamente), muito mais barata que thread de SO: dá pra ter milhares/milhões delas. Diferente do Python:

- **Threading do Python**: limitado pelo GIL pra CPU-bound (só uma thread executa bytecode por vez).
- **asyncio do Python**: concorrência cooperativa, precisa de `async`/`await` explícito em toda a cadeia de chamada, e uma função `async def` só cede controle nos pontos de `await`.
- **Goroutine**: não precisa de `async`/`await` — qualquer função vira concorrente só prefixando `go`, e o scheduler do Go decide quando trocar de contexto (preemptivo desde Go 1.14, não cooperativo puro).

Isso é o maior diferencial de Go mencionado no plano — e a base de todo o trabalho de infra da vaga.

## Channel — comunicação, não memória compartilhada

O idiom de Go pra sincronizar goroutines é "compartilhar memória comunicando", não "comunicar compartilhando memória com lock" (que é o padrão default em Python threading):

```go
ch := make(chan int)   // channel não-bufferizado

go func() {
    ch <- 42   // envia (bloqueia até alguém receber)
}()

v := <-ch   // recebe (bloqueia até alguém enviar)
```

Channel não-bufferizado sincroniza os dois lados (send só completa quando o receive acontece). Channel bufferizado (`make(chan int, 3)`) desacopla até encher o buffer:

```go
ch := make(chan int, 3)
ch <- 1; ch <- 2; ch <- 3   // não bloqueia, buffer tem espaço
```

## Fechamento de channel e range

```go
close(ch)                  // sinaliza "não vem mais nada" — só quem envia deve fechar
for v := range ch {         // lê até o channel fechar
    fmt.Println(v)
}
```

Ler de channel fechado não dá erro — retorna zero value imediatamente. `v, ok := <-ch` com `ok == false` é como saber que fechou sem depender de um valor sentinela.

## Duas armadilhas clássicas (sem equivalente direto em Python)

**Goroutine leak**: disparar uma goroutine que fica bloqueada pra sempre esperando um channel que nunca vai receber/enviar — ela nunca termina, nunca é coletada pelo GC enquanto o processo roda. Não tem exceção pra isso, é silencioso; ferramentas como `pprof` (Módulo 3) ajudam a detectar.

**Variável de loop capturada por closure** — historicamente um bug clássico:

```go
for _, v := range items {
    go func() { fmt.Println(v) }()   // pré-Go 1.22: todas as goroutines viam o MESMO v final
}
```

A partir do Go 1.22 (a versão instalada aqui é 1.26, então isso já está corrigido), cada iteração do `for` tem sua própria cópia de `v` — o bug não existe mais por padrão. Vale saber que existiu, porque código antigo/tutorial pré-1.22 ainda mostra o workaround manual (`v := v` dentro do loop).
