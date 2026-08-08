# net/http e APIs REST

## O que é uma API, do zero

**API** significa "Application Programming Interface" — uma interface de programação de aplicações. Na prática, é um conjunto de portas de entrada que um programa expõe para que **outros programas** (não pessoas, diretamente) consigam pedir alguma coisa a ele: buscar um dado, criar um registro, disparar uma ação.

Pense assim: um site que você usa no navegador tem uma interface pra **pessoas** (botões, formulários, texto na tela). Por trás desse site, existe quase sempre um outro programa — geralmente chamado de **servidor** — que guarda os dados de verdade e faz o trabalho pesado. O navegador (o **cliente**) conversa com esse servidor através de uma API: "me dá a lista de pedidos do usuário 42", "cria um pedido novo com estes itens", etc. Um aplicativo de celular, outro sistema da empresa, ou até um script automatizado também podem ser "clientes" dessa mesma API — API é feita pra ser consumida por código, não só por gente olhando uma tela.

## O protocolo HTTP, do zero

A forma mais comum de cliente e servidor conversarem na internet é através de um protocolo chamado **HTTP** (HyperText Transfer Protocol). "Protocolo" aqui só quer dizer: um formato combinado de mensagem que os dois lados entendem, tipo um roteiro de conversa com regras fixas.

Uma troca HTTP tem sempre duas partes:

1. **Requisição (request)** — o cliente manda, contendo:
   - um **método** (a "ação" que ele quer fazer — `GET` para buscar algo, `POST` para criar, `PUT`/`PATCH` para atualizar, `DELETE` para remover)
   - uma **URL** (o "endereço" do recurso que ele quer, ex: `/pedidos/42`)
   - **headers** (metadados da mensagem — ex: "o corpo desta requisição é JSON")
   - opcionalmente um **corpo** (body) — os dados que ele está enviando, por exemplo os campos de um formulário

2. **Resposta (response)** — o servidor devolve, contendo:
   - um **status code** — um número de três dígitos que resume o resultado (`200` deu certo, `201` algo foi criado, `404` não achei esse recurso, `500` deu erro interno no servidor)
   - headers próprios
   - opcionalmente um corpo — os dados de volta

Nenhum dos dois lados "lembra" de conversas anteriores — cada requisição HTTP é isolada e carrega tudo que o servidor precisa saber para respondê-la. Essa característica se chama **stateless** (sem estado): o servidor não guarda memória de sessão entre uma requisição e a próxima, a não ser que ele mesmo implemente isso deliberadamente (ex: guardando um identificador de sessão num banco).

## O que é REST

**REST** é um estilo (um conjunto de convenções, não uma tecnologia ou biblioteca específica) para desenhar APIs em cima de HTTP. A ideia central: cada URL representa um **recurso** — um "substantivo" do seu sistema (`/pedidos`, `/usuarios/42`) — e o **método HTTP** representa a ação sobre esse recurso, em vez de a ação estar embutida na própria URL:

```
GET    /pedidos        → lista os pedidos
GET    /pedidos/42     → busca o pedido com id 42
POST   /pedidos        → cria um pedido novo
PUT    /pedidos/42     → substitui o pedido 42 inteiro
DELETE /pedidos/42     → remove o pedido 42
```

Repare que a URL nunca tem um verbo (`/criarPedido`, `/deletarPedido42`) — o verbo já está no método HTTP. Uma API "RESTful" é, na prática, uma API que segue essa convenção de forma consistente. O tópico [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#design-de-api) detalha convenções adicionais (versionamento, paginação, idempotência) que valem desde o primeiro endpoint que você escrever, não só quando a API cresce.

## O que é JSON, rapidamente

Praticamente toda API REST moderna troca dados no formato **JSON** (JavaScript Object Notation) — um jeito de escrever texto que representa dados estruturados, parecido com um dicionário de chave e valor:

```json
{"id": 42, "nome": "Mesa de escritório", "preco": 350.0}
```

Este tópico usa JSON nos exemplos porque é o formato universal de corpo de requisição/resposta em API REST, mas o assunto "como o Go converte struct para JSON e vice-versa" tem um tópico dedicado, com muito mais detalhe: [JSON e consumo de APIs externas](../json-e-consumo-de-apis-externas/).

## O pacote net/http

Go vem com uma biblioteca padrão (código que já está instalado junto com a linguagem, sem precisar baixar nada — ver [O que é Go](../o-que-e-go/)) chamada `net/http`, que já contém tudo que é necessário para subir um servidor HTTP funcional e para fazer requisições HTTP como cliente. Isso é incomum: em muitos ecossistemas de linguagem, montar um servidor HTTP decente exige uma biblioteca de terceiros desde o primeiro passo. Em Go, dá para montar uma API REST completa usando só `net/http` — frameworks como [Gin](../gin-framework-web/) existem para tirar boilerplate repetitivo depois que o projeto cresce, não porque a stdlib "não sabe" fazer isso.

## Handler: a função que responde a uma requisição

Um **handler** é uma função com uma assinatura específica que o `net/http` sabe chamar toda vez que chega uma requisição:

```go
func meuHandler(w http.ResponseWriter, r *http.Request) {
    // w: por onde você escreve a resposta (status code, headers, corpo)
    // r: contém tudo sobre a requisição que chegou (método, URL, headers, corpo)
}
```

- `w http.ResponseWriter` — uma interface (ver [Structs, métodos e interfaces](../structs-metodos-e-interfaces/)) que representa "o canal de saída" da resposta. Você escreve nela, e o `net/http` cuida de mandar isso de volta para o cliente pela rede.
- `r *http.Request` — um ponteiro (ver [Ponteiros, slices e maps](../ponteiros-slices-e-maps/)) para uma struct que representa a requisição que chegou: método, URL, headers, corpo, etc. É ponteiro porque a requisição pode ser grande (o corpo, por exemplo) e porque handlers só leem dela, nunca precisam de uma cópia própria.

## ServeMux: registrando rotas

`ServeMux` (multiplexer, "roteador" na prática) é o componente que decide qual handler chamar, olhando o método e a URL da requisição que chegou:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /ola", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Olá, mundo!") // escreve texto puro no corpo da resposta
    })

    log.Println("ouvindo em http://localhost:8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err) // encerra o programa se o servidor não conseguir subir (ex: porta ocupada)
    }
}
```

`http.ListenAndServe(":8080", mux)` faz o programa **escutar** (ficar esperando, indefinidamente) conexões chegando na porta `8080` da máquina, delegando cada requisição para o `mux` decidir qual handler roda. "Porta" aqui é um número que identifica, dentro de uma mesma máquina, qual programa deve receber uma conexão de rede que chega — é possível ter vários programas rodando servidores na mesma máquina, cada um numa porta diferente.

Desde o Go 1.22 (o curso usa 1.26), o `http.ServeMux` da própria stdlib ganhou suporte a **roteamento por método + wildcard de caminho** — o formato `"GET /orders/{id}"` no exemplo acima. Antes da versão 1.22, o `ServeMux` só sabia rotear por prefixo de caminho, sem diferenciar método HTTP nem capturar partes variáveis da URL — nesse cenário mais antigo, era quase impossível montar uma API REST decente sem trazer um router de terceiro (`gorilla/mux`, `chi`). Hoje, para uma API pequena ou média, a stdlib sozinha já é uma opção viável de verdade.

## Path params: capturando partes variáveis da URL

Uma rota como `/pedidos/{id}` tem uma parte fixa (`/pedidos/`) e uma parte variável (`{id}`, chamada de **path param**, "parâmetro de caminho"). O `ServeMux` sabe extrair esse valor para você:

```go
mux.HandleFunc("GET /pedidos/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // se a URL foi /pedidos/42, id vale a string "42"
    fmt.Fprintf(w, "pedido pedido: %s\n", id)
})
```

`r.PathValue("id")` sempre devolve uma `string` — se você precisa de um número, precisa converter manualmente (ex: `strconv.Atoi(id)`), porque toda parte da URL chega como texto puro.

## Query params: dados na própria URL

Além de path params, uma URL pode carregar dados depois de um `?`, no formato `chave=valor` separados por `&` — chamados de **query params** (ex: `/pedidos?status=pago&pagina=2`). São mais usados para filtros, paginação e opções, coisas que não identificam um recurso específico como o `id` identifica:

```go
mux.HandleFunc("GET /pedidos", func(w http.ResponseWriter, r *http.Request) {
    status := r.URL.Query().Get("status") // "" (string vazia) se o parâmetro não veio
    if status != "" {
        fmt.Fprintf(w, "filtrando por status: %s\n", status)
        return
    }
    fmt.Fprintln(w, "listando todos os pedidos")
})
```

`r.URL.Query().Get(...)` devolve string vazia quando o parâmetro simplesmente não veio na URL — não existe erro nem exceção aqui, então se você precisa distinguir "não veio" de "veio vazio de propósito", use `r.URL.Query().Has("status")` em vez de comparar com `""`.

## Lendo o corpo da requisição

Requisições `POST`/`PUT`/`PATCH` costumam vir com um corpo — os dados que o cliente está enviando. `r.Body` é um `io.Reader` (um fluxo de bytes que você lê aos poucos, mais sobre isso em tópicos futuros de I/O). Para o caso comum de "o corpo é JSON, e eu quero isso numa struct Go", usa-se `json.NewDecoder(r.Body).Decode(&algo)`:

```go
type NovoPedido struct {
    ClienteID string  `json:"cliente_id"`
    Valor     float64 `json:"valor"`
}

mux.HandleFunc("POST /pedidos", func(w http.ResponseWriter, r *http.Request) {
    var pedido NovoPedido
    if err := json.NewDecoder(r.Body).Decode(&pedido); err != nil {
        http.Error(w, "corpo da requisição não é um JSON válido", http.StatusBadRequest)
        return
    }
    // pedido.ClienteID e pedido.Valor já estão preenchidos aqui
    fmt.Fprintf(w, "pedido recebido para o cliente %s\n", pedido.ClienteID)
})
```

O detalhe de como `encoding/json` decide o que vira o quê (as struct tags como `` `json:"cliente_id"` ``) está detalhado em [JSON e consumo de APIs externas](../json-e-consumo-de-apis-externas/). `http.Error(w, mensagem, statusCode)` é um atalho que já escreve o status code certo **e** a mensagem de erro no corpo, numa única chamada — bom para casos simples de erro.

## Resposta: status code e corpo, e a ordem que importa

Escrever uma resposta HTTP em Go acontece em duas etapas conceituais, e **a ordem entre elas é rígida**:

```go
w.WriteHeader(http.StatusCreated) // 1) manda o status code (aqui, 201)
json.NewEncoder(w).Encode(pedido) // 2) SÓ DEPOIS, escreve o corpo
```

`w.WriteHeader(código)` precisa ser chamado **antes** de qualquer `Write` (ou `Encode`, que escreve por baixo dos panos) no corpo da resposta. Isso acontece porque, no protocolo HTTP, os headers (que incluem o status code) vêm sempre antes do corpo na mensagem — uma vez que você começou a escrever o corpo, os headers já foram enviados pela rede, e não dá mais para "voltar atrás" e trocar o status code.

Se você nunca chamar `WriteHeader` explicitamente, o Go assume `200 OK` automaticamente na primeira vez que você escreve algo no corpo — por isso o exemplo mais simples deste tópico (o handler `GET /ola`) funciona sem chamar `WriteHeader` nenhuma vez. Só precisa chamar explicitamente quando o status não é `200` — como `201 Created` numa criação, ou `404 Not Found` quando o recurso não existe.

Esquecer essa ordem (chamar `WriteHeader` depois de já ter escrito parte do corpo) não gera um erro de compilação — o programa roda, mas o status code que você tentou mandar depois é silenciosamente ignorado, e o Go registra um aviso no log. É um dos erros mais comuns de quem está começando com a stdlib de Go.

## Exemplo completo: uma API de tarefas em memória

Juntando tudo — roteamento por método, path param, query param, corpo JSON e status code — numa API pequena, guardando os dados só em memória (eles somem quando o programa para; um banco de dados de verdade é assunto de [Acesso a dados e context.Context](../acesso-a-dados-e-context/)):

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
    "sync"
    "time"
)

// Task representa uma tarefa da nossa lista de afazeres.
type Task struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Done bool   `json:"done"`
}

// store guarda as tarefas em memória, protegido por um Mutex porque múltiplas
// requisições podem chegar ao mesmo tempo (cada uma numa goroutine própria —
// ver Concorrência — goroutines e channels).
type store struct {
    mu     sync.Mutex
    tasks  map[int]Task
    nextID int
}

func newStore() *store {
    return &store{tasks: make(map[int]Task), nextID: 1}
}

func (s *store) create(name string) Task {
    s.mu.Lock()
    defer s.mu.Unlock()
    t := Task{ID: s.nextID, Name: name}
    s.tasks[t.ID] = t
    s.nextID++
    return t
}

func (s *store) find(id int) (Task, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    t, ok := s.tasks[id]
    return t, ok
}

func (s *store) all(onlyPending bool) []Task {
    s.mu.Lock()
    defer s.mu.Unlock()
    result := make([]Task, 0, len(s.tasks))
    for _, t := range s.tasks {
        if onlyPending && t.Done {
            continue
        }
        result = append(result, t)
    }
    return result
}

func main() {
    s := newStore()
    mux := http.NewServeMux()

    mux.HandleFunc("GET /tasks", func(w http.ResponseWriter, r *http.Request) {
        onlyPending := r.URL.Query().Get("pending") == "true"
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(s.all(onlyPending))
    })

    mux.HandleFunc("GET /tasks/{id}", func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            http.Error(w, "id inválido", http.StatusBadRequest)
            return
        }
        task, ok := s.find(id)
        if !ok {
            http.Error(w, "tarefa não encontrada", http.StatusNotFound)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(task)
    })

    mux.HandleFunc("POST /tasks", func(w http.ResponseWriter, r *http.Request) {
        var body struct {
            Name string `json:"name"`
        }
        if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
            http.Error(w, "JSON inválido", http.StatusBadRequest)
            return
        }
        if body.Name == "" {
            http.Error(w, "campo name é obrigatório", http.StatusBadRequest)
            return
        }
        task := s.create(body.Name)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated) // 201 — precisa vir antes do Encode
        json.NewEncoder(w).Encode(task)
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    log.Println("ouvindo em http://localhost:8080")
    if err := server.ListenAndServe(); err != nil {
        log.Fatal(err)
    }
}
```

## http.Server: por que não usar só ListenAndServe direto

O exemplo acima monta um `*http.Server` explicitamente, em vez de chamar `http.ListenAndServe(":8080", mux)` direto como no primeiro exemplo do tópico. A diferença importa em produção: `http.ListenAndServe` (a função solta, não o método) usa valores padrão que **não têm timeout nenhum** — uma conexão lenta ou maliciosa pode ficar presa indefinidamente, seguran do um recurso do servidor sem necessidade. Configurar `ReadTimeout`/`WriteTimeout` explicitamente num `*http.Server` é uma das recomendações básicas de resiliência (ver [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#resiliência)) — sem timeout em uma conexão de rede, uma dependência lenta pode travar o serviço inteiro.

## Design de recurso — convenções que valem desde o primeiro endpoint

`GET /pedidos`, não `GET /getPedidos`; versionar a API desde o início (`/v1/...`); paginar qualquer lista que possa crescer sem limite; pensar em idempotência em operações que o cliente pode repetir por causa de uma falha de rede. Essas convenções e o porquê de cada uma estão detalhadas em [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#design-de-api) — vale ler antes de desenhar os primeiros endpoints de um projeto real, não só quando a API já cresceu e ficou difícil de mudar.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise net/http e APIs REST`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: pegar o exemplo da API de tarefas acima, rodar com `go run`, e testar as rotas com `curl` (`curl -X POST localhost:8080/tasks -d '{"name":"estudar Go"}'`, depois `curl localhost:8080/tasks`).
