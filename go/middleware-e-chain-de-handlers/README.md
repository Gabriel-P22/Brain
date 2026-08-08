# Middleware e chain de handlers

## O que é um "handler", rapidamente

Em [net/http e APIs REST](../net-http-e-apis-rest/) você viu que uma função como `getOrder(w http.ResponseWriter, r *http.Request)` é o que responde a uma rota. Essa função é um **handler** — algo que sabe responder a uma requisição HTTP. Formalmente, em Go, um handler é qualquer valor que satisfaz a interface `http.Handler`:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

É uma interface de um método só (ver [Structs, métodos e interfaces](../structs-metodos-e-interfaces/) pra como interface funciona em Go) — qualquer tipo com um método `ServeHTTP` já é, automaticamente, um `http.Handler`, sem precisar declarar isso explicitamente em lugar nenhum (satisfação implícita). `http.HandlerFunc` é um "adaptador" da própria stdlib que transforma uma função comum (`func(w, r)`) num `http.Handler` — é o que permite escrever handlers como funções soltas, sem precisar criar um `struct` só pra ter um método.

## Middleware é uma função que recebe um handler e devolve outro

**Middleware** é código que roda *ao redor* de um handler — antes dele, depois dele, ou nos dois momentos — sem o handler original precisar saber que isso está acontecendo. Casos típicos: logar cada requisição, checar autenticação, medir tempo de resposta, recuperar de um erro fatal.

Em Go, a forma idiomática de escrever middleware é uma **função de ordem superior**: uma função que recebe um `http.Handler` como argumento e devolve um novo `http.Handler`, que por dentro chama o original:

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)   // chama o handler original — aqui é onde o "trabalho de verdade" acontece
        log.Printf("%s %s — %v", r.Method, r.URL.Path, time.Since(start))
    })
}

mux := http.NewServeMux()
mux.HandleFunc("GET /orders/{id}", getOrder)

handler := loggingMiddleware(mux)   // "envolve" o mux inteiro num novo handler
http.ListenAndServe(":8080", handler)
```

Repare que `loggingMiddleware` recebe `next` (o handler que ele está envolvendo) e devolve uma *nova* função que sabe fazer duas coisas: rodar código antes de `next.ServeHTTP(...)` (aqui, nada) e rodar código depois (aqui, o `log.Printf`). Isso não é sintaxe especial nem um recurso mágico do framework — é só uma função Go comum que aceita e devolve outra função, algo que já foi visto em [Sintaxe básica](../sintaxe-basica/#funções-são-um-tipo-de-valor): função em Go é um valor como qualquer outro, então "função que devolve função" não exige nenhum conceito novo, só compor o que já existe.

## Encadeando mais de um middleware manualmente

Nada impede de envolver um handler várias vezes, uma camada por vez:

```go
handler := loggingMiddleware(authMiddleware(mux))
// a chamada mais externa (loggingMiddleware) roda primeiro na ENTRADA,
// mas por último na SAÍDA — é uma pilha, não uma fila
```

Isso funciona, mas fica difícil de ler conforme o número de middlewares cresce (parênteses aninhados demais). Um jeito comum de organizar isso sem trazer nenhum framework externo é escrever uma função `Chain` que aplica uma lista de middlewares em ordem:

```go
// Chain aplica middlewares a um handler, na ordem em que foram passados.
// O primeiro da lista é o mais externo (roda primeiro na entrada).
func Chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

handler := Chain(mux, loggingMiddleware, authMiddleware, recoveryMiddleware)
// lê-se na ordem natural: logging por fora, depois auth, depois recovery, depois o mux
```

`middlewares ...func(http.Handler) http.Handler` é uma função **variádica** (aceita zero ou mais argumentos do mesmo tipo, aqui "função que transforma handler em handler") — permite chamar `Chain` com quantos middlewares quiser, sem precisar mudar a assinatura da função.

## recover — o uso legítimo de panic/recover

Retomando de [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/#panicrecover-não-é-tryexcept): `panic` não é o jeito normal de sinalizar erro em Go (erro esperado sempre volta como `error`), mas `panic` pode acontecer de forma inesperada dentro de um handler (por exemplo, um acesso a um índice de slice que não existe, ou uma tentativa de usar um ponteiro `nil`). Sem tratamento nenhum, um `panic` derruba **o processo inteiro** — não só a requisição que causou o problema, mas todas as outras requisições que estavam em andamento naquele momento, em outras goroutines. É exatamente pra evitar isso que existe o middleware de recovery:

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recuperado: %v", err)
                w.WriteHeader(http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

`defer` (visto em [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/#waitgroup--esperar-n-goroutines-terminarem)) garante que essa função anônima roda mesmo se `next.ServeHTTP` der panic no meio do caminho — é o único jeito de "capturar" um panic em Go: `recover()`, chamado de dentro de uma função adiada com `defer`. Se não houve panic, `recover()` devolve `nil` e não faz nada; se houve, ele interrompe o pânico (o processo não morre) e devolve o valor que foi passado pra `panic(...)`.

Isso é exatamente o que o Gin já inclui de fábrica quando você usa `gin.Default()` (junto com logging) — ver [Gin (framework web)](../gin-framework-web/).

## Chain no Gin

O Gin (framework web usado por cima da stdlib, ver [Gin (framework web)](../gin-framework-web/)) já resolve o encadeamento de middleware de forma embutida, sem precisar de uma função `Chain` própria:

```go
r := gin.New()
r.Use(loggingMiddleware, authMiddleware)   // aplica a TODA rota registrada depois desta linha

func authMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatus(http.StatusUnauthorized)   // corta a chain aqui — handler final nunca roda
        return
    }
    c.Next()   // deixa a execução seguir pro próximo da chain
}
```

`c.Next()` e `c.Abort()` deixam explícito, no próprio código do middleware, onde a cadeia continua ou onde ela para — sem `c.Next()`, o Gin segue pra próxima etapa automaticamente ao final da função; com ele, dá pra controlar precisamente o momento em que o próximo passo roda (útil quando você quer código antes *e* depois do resto da chain, como no exemplo de logging acima).

## Ordem importa — e é uma pilha, não uma fila

Middleware registrado primeiro executa primeiro na entrada da requisição, e por último na saída (a resposta "passa" de volta por todos eles, na ordem inversa). Isso implica uma ordem lógica recomendada:

```go
r.Use(recoveryMiddleware)   // 1º: precisa envolver TUDO, senão panic num middleware seguinte também derruba o processo
r.Use(loggingMiddleware)    // 2º: quer registrar toda requisição, mesmo as que falham em auth
r.Use(authMiddleware)       // 3º: checa autenticação antes de qualquer lógica de negócio rodar
// handlers de negócio vêm depois
```

Se `authMiddleware` viesse antes de `recoveryMiddleware`, um panic dentro do próprio `authMiddleware` não seria capturado por ninguém, e derrubaria o processo — por isso `recovery` é sempre a camada mais externa de todas.
