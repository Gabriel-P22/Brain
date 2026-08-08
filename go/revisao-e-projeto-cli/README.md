# Revisão e projeto CLI

Consolidação da Semana 1 — sem conceito novo, é aplicar tudo que já foi visto junto, num programa pequeno de verdade. Recapitulando rapidamente o que cada tópico anterior cobriu (detalhe completo em cada README):

- [Sintaxe básica](../sintaxe-basica/) — setup do módulo, variáveis (`var`, `:=`), tipos, zero value, conversão de tipo explícita, controle de fluxo (`if`, `for`), funções com múltiplos retornos.
- [Structs, métodos e interfaces](../structs-metodos-e-interfaces/) — struct como tipo composto, método via receiver (valor vs. ponteiro), composição via embedding (no lugar de herança), interface satisfeita implicitamente.
- [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) — `&`/`*`, slice como janela sobre um array compartilhado, `append`, map com zero value `nil` e sem ordem de iteração garantida.
- [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) — `error` como interface comum, erro customizado, wrapping com `%w` + `errors.Is`/`errors.As`, `panic`/`recover` não é o jeito normal de tratar erro, visibilidade por caixa da letra do identificador.
- [Gerenciamento de pacotes](../gerenciamento-de-pacotes/) — `go.mod`/`go.sum`, `go get`, `go mod tidy`, versionamento semântico de módulos.
- [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) — `go` disparando uma goroutine, channel bufferizado/não-bufferizado, `close`+`range`, goroutine leak.
- [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/) — `sync.WaitGroup`, `sync.Mutex`, `select`, testes com o pacote `testing`, table-driven tests.

## Por que vale montar um projeto pequeno agora

Ler sobre cada conceito isoladamente é diferente de usar vários deles juntos, decidindo sozinho quando aplicar cada um. Um projeto pequeno de linha de comando (CLI, "command-line interface" — um programa que roda no terminal, sem interface gráfica) é uma forma de fechar a semana forçando essa integração, sem precisar ainda de servidor HTTP nem banco de dados (assuntos do Módulo 3).

## Ideia de projeto: verificador de URLs

Um programa que recebe uma lista de endereços web, verifica cada um (fazendo uma requisição HTTP simples e conferindo se respondeu), fazendo isso **concorrentemente** (todas as URLs verificadas ao mesmo tempo, não uma de cada vez em sequência), e no final imprime um relatório de quais responderam e quais falharam. Esse projeto único já toca praticamente tudo da semana:

- **struct** para representar o resultado de cada verificação (URL, código de status HTTP, erro se houver).
- **goroutine + WaitGroup** para disparar todas as verificações ao mesmo tempo e esperar todas terminarem antes de imprimir o relatório final.
- **channel** (ou um slice protegido por `Mutex`) para as goroutines devolverem o resultado de cada verificação para quem está esperando.
- **error wrapping** para tratar falha de rede (timeout, DNS não resolvido, conexão recusada) sem derrubar o programa inteiro por causa de uma única URL com problema.
- **slice** para acumular os resultados de todas as URLs verificadas.

### O pacote flag — argumentos de linha de comando

A biblioteca padrão de Go tem um pacote chamado `flag`, que lê argumentos passados na linha de comando (o texto que vem depois do nome do programa, quando você roda ele no terminal) sem precisar de nenhuma dependência externa:

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    url := flag.String("url", "", "URL a verificar")     // flag de texto, com valor padrão ""
    timeout := flag.Int("timeout", 5, "timeout em segundos") // flag numérica, com valor padrão 5
    flag.Parse()                                              // lê de verdade os argumentos passados

    fmt.Println("verificando:", *url, "com timeout de", *timeout, "segundos")
}
```

Rodando esse programa assim:

```
go run main.go -url=https://example.com -timeout=10
```

Imprime: `verificando: https://example.com com timeout de 10 segundos`.

Repare que `flag.String` e `flag.Int` devolvem um **ponteiro** (`*string`, `*int`) — por isso o `*url` e o `*timeout` na hora de usar o valor, desreferenciando o ponteiro (assunto explicado em [Ponteiros, slices e maps](../ponteiros-slices-e-maps/#ponteiros--o-endereço-de-uma-variável-na-memória)). Isso é necessário porque, no momento em que você chama `flag.String(...)`, os argumentos ainda não foram lidos de verdade (isso só acontece depois, em `flag.Parse()`) — o pacote `flag` devolve o endereço de uma variável que ele mesmo vai preencher depois, quando `Parse()` rodar.

Também é possível receber várias URLs de uma vez, sem usar `flag` para isso — os argumentos "soltos" (que não são flags nomeadas) ficam disponíveis depois de `flag.Parse()` através de `flag.Args()`:

```go
flag.Parse()
urls := flag.Args() // slice de string com tudo que sobrou depois das flags nomeadas

for _, u := range urls {
    fmt.Println("URL recebida:", u)
}
```

```
go run main.go https://example.com https://golang.org https://naoexiste.invalido
```

### Esboço da lógica concorrente

Um esqueleto de como as peças se encaixam — não é código pronto, é para mostrar a forma geral da solução:

```go
package main

import (
    "fmt"
    "net/http"
    "sync"
    "time"
)

type Resultado struct {
    URL    string
    Status int
    Err    error
}

func verificar(url string) Resultado {
    cliente := http.Client{Timeout: 5 * time.Second}
    resp, err := cliente.Get(url)
    if err != nil {
        return Resultado{URL: url, Err: fmt.Errorf("verificando %s: %w", url, err)}
    }
    defer resp.Body.Close()
    return Resultado{URL: url, Status: resp.StatusCode}
}

func main() {
    urls := []string{
        "https://example.com",
        "https://golang.org",
        "https://naoexiste.invalido",
    }

    var wg sync.WaitGroup
    var mu sync.Mutex
    resultados := make([]Resultado, 0, len(urls))

    for _, u := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            r := verificar(u)

            mu.Lock()
            resultados = append(resultados, r)
            mu.Unlock()
        }(u)
    }

    wg.Wait()

    for _, r := range resultados {
        if r.Err != nil {
            fmt.Println(r.URL, "- FALHOU:", r.Err)
            continue
        }
        fmt.Println(r.URL, "- status", r.Status)
    }
}
```

Esse esboço usa `sync.Mutex` para proteger o slice `resultados` compartilhado entre todas as goroutines (mesmo padrão explicado em [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/#mutex--quando-duas-goroutines-precisam-mexer-no-mesmo-dado)) — uma alternativa igualmente válida seria usar um channel bufferizado para cada goroutine enviar seu `Resultado`, e uma única goroutine central consumindo esse channel para montar o slice, sem precisar de `Mutex` nenhum. Vale como exercício pensar nas duas formas e comparar.

## Estrutura de pastas sugerida

Mesmo num projeto pequeno, já vale separar "a lógica que verifica uma URL" (que não depende de terminal nem de rede real — pode ser testada isoladamente) de "o programa que lê argumentos de linha de comando e chama essa lógica". Essa separação é uma primeira aplicação, ainda informal, de organização em camadas — assunto que volta bem mais a fundo no Módulo 2, em [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/):

```
projeto-cli/
  main.go          // lê as flags, orquestra a chamada, imprime o relatório final
  checker/
    checker.go      // lógica pura de "verificar uma URL" — testável sem precisar de rede real de verdade
    checker_test.go // testes dessa lógica, com um cliente HTTP substituível
```

Separar assim traz uma vantagem concreta e imediata: `checker.go` pode ser testado com `go test` sem precisar acessar a rede de verdade a cada execução de teste (usando um `http.Client` configurável, ou uma interface pequena no lugar dele) — o mesmo raciocínio de testabilidade via interface pequena que aparece, de forma mais completa, no Módulo 2 e no Módulo 3 deste curso.

## Próximo passo

Este projeto não vem com um exercício pré-gerado — quando quiser montar de verdade, rode `/exercise revisão e projeto cli`, ou peça o esboço direto no chat. A ideia aqui é ter a arquitetura clara antes de escrever a primeira linha: struct de resultado, goroutine por URL, WaitGroup pra sincronizar, error wrapping por falha individual, flag pra entrada de linha de comando.
