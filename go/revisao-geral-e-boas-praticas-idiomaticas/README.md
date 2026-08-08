# Revisão geral e boas práticas idiomáticas

Fechamento do plano de 2 semanas: não introduz conceito novo, consolida os 26 tópicos anteriores (Módulos 1-3) num formato de consulta rápida — recap de armadilhas, checklist idiomático e perguntas típicas de dia-1/entrevista técnica. Cada item aqui vem com um trecho de código curto ilustrando o problema (ou a prática), não só a frase-resumo — a ideia é que este documento sirva pra revisar sozinho, sem precisar reabrir todos os tópicos anteriores.

## Recap por módulo

### Módulo 1 — Fundamentos

Tipagem estática (o tipo de cada variável é fixo e checado antes do programa rodar), variável sem valor definido começa em "zero value" (nunca fica indefinida), ponteiro explícito (`*T`/`&valor`), slice compartilha o array por baixo do original, `error` é um valor de retorno normal (não uma exceção que interrompe o fluxo), goroutines/channels para concorrência, `go.mod`/`go.sum` para dependências.

Um exemplo pequeno que junta vários desses fundamentos numa função só:

```go
// buscarIdade procura o nome no mapa e devolve a idade encontrada.
// Se o nome não existir, devolve um erro em vez de um valor "mágico" (como -1).
func buscarIdade(idades map[string]int, nome string) (int, error) {
    idade, existe := idades[nome] // "existe" vem do próprio acesso ao map — segundo valor de retorno
    if !existe {
        return 0, fmt.Errorf("nome %q não encontrado no mapa", nome)
        // 0 aqui é só o zero value de int, devolvido por convenção — quem chama
        // NUNCA deve olhar pra esse 0 sem checar o erro antes
    }
    return idade, nil
}

idade, err := buscarIdade(map[string]int{"Ana": 30}, "Bruno")
if err != nil {
    fmt.Println("erro:", err) // erro: nome "Bruno" não encontrado no mapa
    return
}
fmt.Println("idade:", idade)
```

Detalhe: `idades[nome]` sozinho (sem o `, existe`) também funciona e devolve só o valor — mas aí, se a chave não existir, você recebe silenciosamente o zero value (`0`) sem saber se era porque a idade real era zero ou porque a chave não existia. Pedir os dois valores de retorno (`idade, existe := ...`) é o jeito idiomático de distinguir "a chave existe e vale zero" de "a chave não existe".

### Módulo 2 — SOLID / DDD / Clean Architecture / Clean Code

Os mesmos princípios de sempre (definidos de forma independente de linguagem em `contexts/common/{SOLID,DDD,CLEAN-ARCHITECTURE,CLEAN-CODE}.md`), só que expressos do jeito Go: interface pequena e descoberta por quem consome (não declarada de antemão por quem implementa), composição via **embedding** de struct no lugar de herança (Go não tem herança de classe — é uma decisão deliberada de design, não uma limitação), camadas separadas por pacote, e a pasta especial `internal/` para impedir que código de fora do módulo importe pacotes que deveriam ser detalhe interno.

Você vai ver exemplos concretos de interface pequena (Interface Segregation) e de embedding (composição) nas seções de checklist e perguntas frequentes mais abaixo — aqui vale só fixar o resumo: em Go, esses princípios não são "regra a aplicar por cima", eles nascem quase naturalmente da forma como a linguagem é desenhada (interfaces implícitas, sem herança, pacotes como fronteira física de código).

### Módulo 3 — Backend em Go

`net/http`/Gin para servir HTTP, JSON e struct tags para (de)serializar dados, `context.Context` para propagar cancelamento/timeout por uma cadeia de chamadas, repository pattern para isolar acesso a dados, middleware para lógica transversal (log, autenticação, etc. aplicada a várias rotas de uma vez), worker pools/rate limiting para controlar concorrência em produção, logging estruturado, testes table-driven.

Um esqueleto de handler HTTP que amarra vários desses pontos:

```go
// BuscarUsuario é um handler HTTP: recebe a requisição, delega pro repositório,
// e traduz o resultado (ou erro) em resposta HTTP.
func (h *Handler) BuscarUsuario(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()           // contexto da requisição, cancelado se o cliente desconectar
    id := r.PathValue("id")      // parâmetro de rota (net/http puro, Go 1.22+)

    usuario, err := h.repo.Buscar(ctx, id) // ctx propagado até a camada de dados
    if err != nil {
        http.Error(w, "usuário não encontrado", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(usuario)
}
```

Detalhe de cada um continua só na `README.md` do respectivo tópico — este arquivo não repete a explicação completa, só aponta o que revisar se algo não estiver firme.

## Armadilhas mais comuns

Já explicadas no tópico onde surgiram — aqui a tabela funciona como índice rápido ("se esquecer, foi isso"), e logo abaixo tem um trecho de código pra cada uma, mostrando o problema na prática.

| # | Armadilha | Onde foi coberta |
|---|---|---|
| 1 | Slice compartilha array subjacente do original (`append` pode ou não realocar) | [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) |
| 2 | Map com zero value `nil` — leitura ok, escrita causa panic | [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) |
| 3 | Ignorar `error` silenciosamente (nem sempre precisa de `_ =` pra isso) | [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) |
| 4 | Goroutine leak — goroutine bloqueada pra sempre, sem exceção que avise | [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) |
| 5 | Receiver ponteiro vs valor inconsistente no mesmo tipo (mistura gera bug sutil de cópia) | [Structs, métodos e interfaces](../structs-metodos-e-interfaces/) |
| 6 | `go.sum` esquecido fora do commit (nunca deveria ser gitignored) | [Gerenciamento de pacotes](../gerenciamento-de-pacotes/) |
| 7 | `context.Context` não propagado numa chamada bloqueante — timeout do caller não corta a chamada de baixo | [Acesso a dados e context.Context](../acesso-a-dados-e-context/) |

### 1. Slice compartilha array subjacente

Um "slice" não é uma cópia de dados — é uma "janela" (ponteiro + tamanho + capacidade) sobre um array que existe em algum lugar da memória. Fatiar um slice (`slice[a:b]`) não copia nada, só cria outra janela apontando pro mesmo array:

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:3] // sub = [2, 3], mas aponta pro MESMO array de memória que original

sub[0] = 99
fmt.Println(original) // [1 99 3 4 5] — mudar sub mudou original, porque compartilham o array

sub = append(sub, 100) // ainda cabe no array original (tem espaço sobrando na capacidade)
fmt.Println(original) // [1 99 3 100 5] — o "4" virou "100" sem ninguém escrever em "original" diretamente
```

O jeito de evitar essa surpresa, quando você precisa mesmo de uma cópia independente, é usar `copy`:

```go
copiaIndependente := make([]int, len(original))
copy(copiaIndependente, original)
copiaIndependente[0] = -1
fmt.Println(original[0]) // continua 1 — não foi afetado
```

### 2. Map com zero value `nil`

O zero value de um `map` não declarado é `nil` — e um map `nil` deixa você **ler** dele (devolve o zero value do tipo do valor), mas causa **panic** (o programa trava com erro fatal) se você tentar **escrever** nele:

```go
var m map[string]int   // zero value de map: nil (nenhum map de verdade foi criado ainda)
fmt.Println(m["chave"]) // 0 — ler de um map nil funciona, devolve o zero value do tipo int
// m["chave"] = 1       // panic: assignment to entry in nil map

m2 := make(map[string]int) // isto sim cria um map vazio, pronto pra escrita
m2["chave"] = 1             // ok, funciona normalmente
fmt.Println(m2)             // map[chave:1]
```

A regra prática: sempre que você for **escrever** em um map (não só ler), garanta que ele foi criado com `make(map[K]V)` ou com um literal (`map[string]int{}`), nunca deixado no zero value.

### 3. Ignorar `error` silenciosamente

Go não obriga você a usar o valor de retorno de uma função. Isso significa que dá pra ignorar um erro sem sequer escrever `_ =` — o compilador não reclama:

```go
func escreverArquivo(caminho string, dados []byte) error {
    return os.WriteFile(caminho, dados, 0644)
}

func salvar() {
    escreverArquivo("saida.txt", []byte("dados")) // erro totalmente ignorado, e compila normal
    fmt.Println("salvo!") // essa mensagem aparece mesmo se a escrita tiver falhado
}
```

O jeito correto é sempre capturar e checar:

```go
func salvar() error {
    if err := escreverArquivo("saida.txt", []byte("dados")); err != nil {
        return fmt.Errorf("salvando arquivo: %w", err)
    }
    fmt.Println("salvo!")
    return nil
}
```

Ferramentas como `golangci-lint` (linter `errcheck`) pegam esse tipo de erro ignorado automaticamente — é por isso que rodar o linter antes de commitar (ver seção de ferramentas abaixo) é tão importante em Go: o compilador sozinho não cobre esse caso.

### 4. Goroutine leak

Uma goroutine "vazada" é uma goroutine que fica presa esperando algo que nunca vai acontecer — ela nunca é liberada da memória, e não existe nenhum aviso automático de que isso aconteceu (diferente de uma exceção não tratada, que pelo menos aparece bem visível):

```go
func vazamento() {
    ch := make(chan int) // channel sem buffer: só recebe se alguém estiver esperando
    go func() {
        valor := <-ch // fica esperando pra sempre — ninguém nunca envia nada em ch
        fmt.Println(valor)
    }()
    // a função "vazamento" termina aqui, mas a goroutine acima continua viva,
    // presa nesse <-ch, ocupando memória indefinidamente
}
```

O jeito de evitar: sempre ter um caminho de saída, geralmente com `context.Context` cancelável:

```go
func semVazamento(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case valor := <-ch:
            fmt.Println(valor)
        case <-ctx.Done(): // se o contexto for cancelado, a goroutine termina de qualquer jeito
            fmt.Println("cancelado, saindo")
            return
        }
    }()
}
```

### 5. Receiver ponteiro vs valor inconsistente

Um método em Go pode ter "receiver" (o `c` antes do nome do método) por valor ou por ponteiro. Por valor, o método recebe uma **cópia** da struct; por ponteiro, ele recebe o **endereço** da struct original. Misturar os dois no mesmo tipo é uma fonte clássica de bug sutil:

```go
type Contador struct {
    valor int
}

func (c Contador) Incrementar() { // receiver por VALOR — c aqui é uma cópia
    c.valor++ // incrementa a cópia; o Contador original não muda
}

func (c *Contador) IncrementarCerto() { // receiver por PONTEIRO — c aqui é o endereço do original
    c.valor++ // incrementa o valor de verdade
}

func main() {
    c := Contador{valor: 0}

    c.Incrementar()
    fmt.Println(c.valor) // 0 — nada mudou, Incrementar só alterou uma cópia descartada

    c.IncrementarCerto()
    fmt.Println(c.valor) // 1 — agora sim, porque o método trabalhou no endereço real
}
```

A convenção idiomática é escolher **um** estilo de receiver por tipo e manter consistente em todos os métodos daquele tipo — normalmente ponteiro, se qualquer um dos métodos precisa modificar o estado ou se a struct é grande (copiar fica caro).

### 6. `go.sum` fora do commit

`go.sum` guarda o hash (uma "impressão digital" criptográfica) de cada dependência baixada — direta e indireta. Sem ele versionado, dois desenvolvedores (ou o CI) podem acabar resolvendo dependências de formas diferentes, ou sem checagem nenhuma de integridade:

```
# .gitignore ERRADO — nunca faça isso em um projeto Go:
go.sum

# certo: go.mod E go.sum sempre no repositório, ambos versionados
```

O tópico [Gerenciamento de pacotes](../gerenciamento-de-pacotes/) detalha a diferença entre `go.mod` (o que) e `go.sum` (a garantia de que é exatamente aquilo).

### 7. `context.Context` não propagado

Se uma função bloqueante (uma chamada de rede, uma query de banco) não recebe o `context.Context` da chamada que a originou, cancelar/dar timeout lá em cima não corta essa chamada de baixo — ela continua rodando até terminar sozinha, mesmo que quem pediu já tenha desistido de esperar:

```go
// ERRADO: buscarUsuarioSemCtx não recebe contexto nenhum
func buscarUsuarioSemCtx(id string) (*Usuario, error) {
    return db.Query("SELECT * FROM usuarios WHERE id = ?", id) // roda até o fim, sem jeito de cancelar
}

// CERTO: contexto propagado desde o handler HTTP até a query
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // já vem com cancelamento automático se o cliente desconectar
    usuario, err := buscarUsuarioComCtx(ctx, id)
    // ...
}

func buscarUsuarioComCtx(ctx context.Context, id string) (*Usuario, error) {
    return db.QueryContext(ctx, "SELECT * FROM usuarios WHERE id = ?", id) // cancela junto com ctx
}
```

## Checklist idiomático rápido

Antes de considerar um trecho de Go "pronto", pergunte (e veja o exemplo de cada um):

### Erro tem contexto?

```go
// evite: erro "nu", devolvido sem dizer onde ele aconteceu
func lerConfig(caminho string) (*Config, error) {
    dados, err := os.ReadFile(caminho)
    if err != nil {
        return nil, err // quem recebe isso não sabe que veio de "lerConfig" nem de qual caminho
    }
    return parse(dados)
}

// idiomático: erro envolvido (wrapped) com contexto de onde e o quê
func lerConfig(caminho string) (*Config, error) {
    dados, err := os.ReadFile(caminho)
    if err != nil {
        return nil, fmt.Errorf("lendo config em %s: %w", caminho, err)
    }
    return parse(dados)
}
```

O `%w` (em vez de `%v` ou `%s`) é o que faz Go "envolver" o erro original preservando-o — quem recebe esse erro ainda consegue checar o erro original por dentro com `errors.Is`/`errors.As`, mesmo depois de várias camadas de wrapping. Ver [Clean Code em Go](../clean-code-em-go/) para o detalhe completo.

### Interface está do lado de quem consome, não de quem implementa?

```go
// evite: interface enorme, declarada no pacote que implementa, "por via das dúvidas",
// listando tudo que a struct sabe fazer
type Repositorio interface {
    Buscar(id string) (*Usuario, error)
    Salvar(u *Usuario) error
    Deletar(id string) error
    ListarTodos() ([]*Usuario, error)
    ContarPorStatus(status string) (int, error)
}

// idiomático: quem consome declara só o que precisa, no próprio pacote que consome
package servico

type buscadorDeUsuario interface {
    Buscar(id string) (*Usuario, error) // só isso — é só isso que BuscarNomeDoUsuario usa
}

func BuscarNomeDoUsuario(b buscadorDeUsuario, id string) (string, error) {
    u, err := b.Buscar(id)
    if err != nil {
        return "", err
    }
    return u.Nome, nil
}
```

Interface pequena e do lado do consumidor é a versão em Go de dois princípios de SOLID ao mesmo tempo: Interface Segregation (ninguém deveria ser forçado a depender de método que não usa) e Dependency Inversion (quem consome define a abstração que precisa, quem implementa nem precisa saber que a interface existe). Ver [SOLID em Go](../solid-em-go/) e `go/config/reference/SOLID.md`.

### `gofmt`/`goimports` rodaram?

```
gofmt -l .
# saída vazia = tudo formatado no padrão oficial
# se aparecer algum caminho de arquivo na saída, esse arquivo está fora do padrão

gofmt -w .
# corrige a formatação de todos os arquivos automaticamente, sem precisar decidir nada manualmente
```

Não é opinião de estilo, é automático — se o CI reclamar de formatação é porque isso não rodou antes do commit.

### `golangci-lint run` limpo?

```
golangci-lint run
# ./servico/usuario.go:42:2: err113: do not compare against errors directly (errorlint)
# ./repositorio/postgres.go:18:1: unused-parameter: parameter 'ctx' is unused (revive)
```

`golangci-lint` agrega dezenas de analisadores num só comando — cobre boa parte do que, sem essa ferramenta, seria comentário manual de code review (variável declarada e nunca usada, sombra de variável com o mesmo nome de outra em escopo externo, erro ignorado sem indicação explícita, comparação de erro por igualdade em vez de `errors.Is`, etc).

### Pacote tem um propósito claro?

```go
// evite: pacote genérico virando "gaveta de miscelânea"
package utils

func FormatarData(t time.Time) string { /* ... */ }
func ValidarCPF(cpf string) bool      { /* ... */ }
func EnviarEmail(destino, assunto string) error { /* ... */ }
// três responsabilidades sem nenhuma relação entre si, empilhadas só porque "sobrou"

// idiomático: um pacote por responsabilidade coesa
package datafmt
func Formatar(t time.Time) string { /* ... */ }

package validacao
func CPF(cpf string) bool { /* ... */ }

package email
func Enviar(destino, assunto string) error { /* ... */ }
```

Nome de pacote genérico (`utils`, `common`, `helpers`, `misc`) é sinal de Single Responsibility violado no nível de organização do código, não só no nível de uma função ou tipo isolado — o pacote deveria ter um propósito que dá pra descrever em uma frase curta.

### Concorrência tem forma clara de terminar?

```go
func processarTodos(ctx context.Context, itens []int) {
    var wg sync.WaitGroup

    for _, item := range itens {
        wg.Add(1)
        go func(item int) {
            defer wg.Done() // garante que o WaitGroup sabe que essa goroutine terminou
            select {
            case <-ctx.Done():
                return // encerra cedo se o contexto foi cancelado
            default:
                processar(item)
            }
        }(item)
    }

    wg.Wait() // bloqueia até TODAS as goroutines disparadas acima terminarem
}
```

Toda goroutine disparada precisa de um caminho claro de retorno — `context` cancelável, `sync.WaitGroup`, ou um channel fechado que sinalize "pode parar". "Disparei com `go` e nunca mais pensei nisso" é exatamente o padrão que causa o goroutine leak da armadilha #4 acima.

## Perguntas frequentes de dia-1 / entrevista técnica

### Por que Go não tem `try`/`except`?

Erro como valor de retorno força quem chama a função a decidir explicitamente o que fazer com aquele erro — não existe um "caminho de erro" silencioso subindo pilha acima sem estar declarado na própria assinatura da função (o `error` aparece ali, no tipo de retorno, visível pra qualquer um que leia a assinatura).

```go
// fluxo normal de erro esperado: tratado explicitamente, no lugar onde acontece
resultado, err := dividir(10, 0)
if err != nil {
    fmt.Println("não foi possível dividir:", err)
    return
}
```

Go tem `panic`/`recover`, mas isso é reservado pra estado realmente inconsistente (um bug interno, uma situação que não deveria acontecer de jeito nenhum) — não é o mecanismo usado para fluxo de erro esperado do dia a dia, como "arquivo não encontrado" ou "usuário inválido":

```go
func processarComSeguranca() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recuperado: %v", r) // transforma um panic em erro normal
        }
    }()
    return operacaoArriscada() // se isso der panic, o defer acima "pega" e converte pra erro
}
```

### Qual a diferença entre array e slice?

```go
var arr [5]int // array: o tamanho faz parte do próprio TIPO — [5]int é um tipo diferente de [10]int
arr[0] = 1
// arr = append(arr, 2) // nem compila: array não tem append, tamanho é fixo pra sempre

s := []int{1, 2, 3} // slice: tamanho NÃO faz parte do tipo — pode crescer
s = append(s, 4)
fmt.Println(len(s), cap(s)) // len = quantidade de elementos atual; cap = quanto cabe antes de realocar
```

Array puro é raro no código do dia a dia em Go — quase todo código usa slice, que é a estrutura dinâmica de verdade (por baixo: um ponteiro pra um array + um tamanho + uma capacidade).

### Quando usar goroutine vs quando não usar?

```go
// não vale a pena: trabalho pequeno e sequencial — o overhead de criar/coordenar
// a goroutine custa mais do que o trabalho em si
resultado := somar(2, 3)

// vale a pena: I/O independente que pode rodar em paralelo
resultados := make(chan int, len(urls))
for _, url := range urls {
    go func(u string) {
        resultados <- buscar(u) // uma chamada de rede lenta não bloqueia as outras
    }(url)
}
for range urls {
    fmt.Println(<-resultados)
}
```

A regra prática: dispare goroutine quando há trabalho que pode rodar de forma independente (tipicamente esperando algo externo — rede, disco) e o custo de coordenação (channel, `sync`) compensa o ganho. Para trabalho puramente sequencial e rápido, `go` só adiciona overhead de agendamento sem ganho nenhum.

### O que `context.Context` resolve que um timeout isolado não resolve?

```go
// um timeout isolado, só na função de topo, NÃO se propaga:
func handlerErrado(w http.ResponseWriter, r *http.Request) {
    time.AfterFunc(2*time.Second, func() { /* cancela só isso aqui, não o resto */ })
    dados, _ := repositorio.BuscarSemContexto(id) // continua rodando, ignorando o timeout de cima
}

// context.Context propaga cancelamento/deadline por toda a cadeia de chamadas:
func handlerCerto(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    dados, err := repositorio.Buscar(ctx, id) // handler → service → repository → driver de banco,
    // se qualquer ponto dessa cadeia checar ctx.Done(), o cancelamento do topo propaga até lá
}
```

A palavra-chave é propagação: um único `context` carregado como primeiro parâmetro de cada função ao longo de toda a cadeia de chamadas garante que "cortar" no topo (a requisição HTTP foi cancelada) corta automaticamente qualquer chamada bloqueante feita nas camadas de baixo.

### Por que Go não tem herança?

Decisão deliberada de design: composição (embedding de struct ou de interface dentro de outra struct) resolve reuso de código sem a fragilidade de hierarquias profundas de classes. Em Go isso não é uma escolha de estilo entre várias opções — é a única forma que a linguagem oferece.

```go
type Animal struct {
    Nome string
}

func (a Animal) Descricao() string {
    return "Animal chamado " + a.Nome
}

type Cachorro struct {
    Animal // embedding: Cachorro "ganha" os campos e métodos de Animal automaticamente
    Raca string
}

c := Cachorro{Animal: Animal{Nome: "Rex"}, Raca: "Labrador"}
fmt.Println(c.Descricao()) // "Animal chamado Rex" — método promovido de Animal, sem reescrever nada
fmt.Println(c.Nome)        // "Rex" — campo promovido também
```

A diferença crucial pra herança clássica: `Cachorro` **tem um** `Animal` embutido (composição), ele não **é um** `Animal` no sentido de hierarquia de tipos — não existe um "tipo base" `Animal` do qual `Cachorro` "descende" formalmente para fins de polimorfismo. Se você precisa que `Cachorro` seja tratado como algo genérico junto com outros tipos, isso é resolvido com **interface**, não com embedding. Ver [DDD em Go](../ddd-em-go/) e `go/config/reference/DDD.md` para composição aplicada a modelagem de domínio.

### O que `go.sum` garante que `go.mod` sozinho não garante?

```
go.mod:
    module meuprojeto
    require github.com/exemplo/lib v1.2.0

go.sum:
    github.com/exemplo/lib v1.2.0 h1:AbCdEf1234567890fedcba...=
    github.com/exemplo/lib v1.2.0/go.mod h1:XyZ9876543210abc...=
```

`go.mod` diz **qual versão** de cada dependência o projeto usa. `go.sum` trava o **hash** (impressão digital criptográfica) de cada dependência baixada — direta e indireta (transitiva) — de forma que um build não pode silenciosamente puxar um artefato alterado que tenha, por algum motivo, o mesmo número de versão mas conteúdo diferente do esperado.

## Ferramentas — rotina prática, sem re-explicar cada uma

`gofmt`, `goimports` e `golangci-lint` já foram cobertos como convenção automática em [Clean Code em Go](../clean-code-em-go/) e no exemplo idiomático em [go/config/reference/CLEAN-CODE.md](../config/reference/CLEAN-CODE.md). Rotina prática antes de qualquer commit, na ordem:

```
gofmt -l .            # lista arquivo fora do padrão de formatação (saída vazia = tudo ok)
go vet ./...           # erros comuns que compilam mas são suspeitos (ex: Printf com argumento errado)
golangci-lint run       # agrega múltiplos linters de uma vez (inclui vet, staticcheck, errcheck, etc)
go test ./...            # roda toda a bateria de testes do módulo
```

Exemplo do tipo de coisa que `go vet` pega, que compila normalmente mas está errado:

```go
nome := "Ana"
fmt.Printf("Olá, %d\n", nome) // %d espera número, "nome" é string — compila, mas roda errado
// go vet ./... avisa: Printf format %d has arg nome of wrong type string
```

Rodar esses quatro comandos, nessa ordem, antes de qualquer commit é hábito suficiente para pegar a grande maioria dos problemas cobertos nas armadilhas listadas acima, antes mesmo de alguém revisar o código manualmente.

## Gaps conhecidos — o que este plano não cobriu a fundo

Honestidade sobre o que ficou de fora das 2 semanas (não é obrigatório pra dia-1, mas vale saber que existe e ter uma ideia mínima do que é):

- **Generics** (`[T any]`, desde Go 1.18) — permitem escrever uma função ou tipo que funciona para vários tipos diferentes, sem duplicar código nem usar `interface{}`/`any` (que perde a checagem de tipo em tempo de compilação):

```go
// Contem funciona pra qualquer tipo "comparável" (que suporta ==), sem duplicar a função
func Contem[T comparable](lista []T, alvo T) bool {
    for _, item := range lista {
        if item == alvo {
            return true
        }
    }
    return false
}

Contem([]int{1, 2, 3}, 2)        // funciona com int
Contem([]string{"a", "b"}, "a")  // e com string, mesma função, sem reescrever nada
```

  Útil saber que existe e a sintaxe básica (`[T any]`, `[T comparable]`), mas a maior parte do código de backend do dia a dia ainda é escrita sem generics — a maioria dos problemas resolvidos com interface pequena já cobre o que generics resolveria em código de aplicação comum.

- **Reflection** (pacote `reflect`) — permite que um programa inspecione o tipo/estrutura de um valor em tempo de execução. Usado internamente por bibliotecas (o pacote `encoding/json`, por exemplo, usa reflection pra descobrir os campos de uma struct via struct tags), mas raramente escrito à mão em código de aplicação.
- **cgo / interop com C** — mecanismo para chamar código C de dentro de um programa Go. Irrelevante pro perfil de API/infra do cargo.
- **Build tags e cross-compilation avançada** — as variáveis `GOOS`/`GOARCH` (que controlam pra qual sistema operacional/arquitetura o `go build` gera o binário) foram mencionadas de passagem em [O que é Go](../o-que-e-go/), sem aprofundar em matriz de build multi-plataforma ou build tags condicionais (`//go:build linux`, por exemplo).

Se alguma dessas aparecer em entrevista, é razoável responder "sei que existe e pra que serve, não usei na prática ainda" — resposta honesta de alguém que fez 2 semanas de curva de aprendizado, não uma lacuna pra esconder.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise revisão geral`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Se quiser um exercício de fechamento de verdade, o candidato natural é revisitar o projeto proposto em [Revisão e projeto CLI](../revisao-e-projeto-cli/) ou o esboço de [Projeto final — API + LLM](../projeto-final-api-llm/), já que nenhum dos dois foi implementado ainda.
