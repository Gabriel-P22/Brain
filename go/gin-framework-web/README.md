# Gin (framework web)

## O que é um "framework web"

Um **framework** é uma biblioteca de terceiros (não faz parte da linguagem nem da biblioteca padrão) que já resolve, de fábrica, um conjunto de problemas recorrentes de um certo tipo de aplicação — nesse caso, aplicações web. [net/http e APIs REST](../net-http-e-apis-rest/) mostrou como montar uma API só com a biblioteca padrão do Go (`net/http`); um framework como **Gin** entra por cima disso, adicionando conveniências que você teria que escrever manualmente na mão com a stdlib pura.

Gin é o framework web mais usado no ecossistema Go hoje. Ele não substitui `net/http` — por baixo dos panos, Gin ainda roda em cima do modelo de servidor HTTP do Go, só adiciona uma camada de conveniência por cima.

## Por que sair da stdlib

`net/http` com o `ServeMux` moderno (Go 1.22+) já resolve roteamento básico bem. Gin compensa quando o projeto precisa, de fábrica, de coisas que a stdlib deixa para você implementar manualmente:

- **Decodificar e validar o corpo da requisição num único passo** — na stdlib, você decodifica o JSON manualmente e depois valida os campos manualmente, em dois passos separados; Gin faz isso junto, usando anotações no próprio struct.
- **Ecossistema grande de middleware pronto** — autenticação, CORS, compressão, rate limiting, logging estruturado — bibliotecas de terceiros compatíveis com Gin, prontas para usar.
- **Performance de roteamento** — o router interno do Gin usa uma estrutura de dados chamada radix tree, otimizada para casar URLs rapidamente mesmo com centenas de rotas registradas.
- **Agrupamento de rotas** com prefixo e middleware compartilhado (ver mais abaixo).

Para uma API pequena, a stdlib sozinha resolve bem — não é obrigatório usar framework nenhum. A decisão de trazer Gin (ou qualquer outra dependência externa) vale a pena quando o ganho de produtividade supera o custo de mais uma dependência para manter atualizada.

## Instalando

Gin é uma dependência externa — precisa ser baixada e registrada no `go.mod` do projeto (ver [Gerenciamento de pacotes](../gerenciamento-de-pacotes/)):

```
go get github.com/gin-gonic/gin
```

Isso adiciona uma linha em `go.mod` com o caminho do módulo e a versão baixada, e cria/atualiza o `go.sum` (o arquivo que trava os hashes de tudo que foi baixado, para builds reprodutíveis).

## Servidor mínimo

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // já vem com dois middlewares: logger de requisições + recovery de panic

    r.GET("/ola", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"mensagem": "olá, mundo!"})
    })

    r.Run(":8080") // equivalente a http.ListenAndServe(":8080", r), mas mais curto
}
```

`gin.Default()` cria um roteador Gin já com dois middlewares ativados por padrão: um que registra no log cada requisição recebida (método, caminho, status, tempo de resposta), e outro que captura panics dentro de um handler e responde `500 Internal Server Error` em vez de derrubar o servidor inteiro (mais sobre panic/recover em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)). Se você não quiser esses dois de graça, existe `gin.New()`, que cria um roteador vazio.

`gin.H` é só um apelido (um `type` declarado dentro do próprio pacote Gin) para `map[string]interface{}` — uma forma curta de escrever um mapa de chave string para qualquer valor, útil para montar respostas JSON ad-hoc sem precisar declarar uma struct para cada formato de resposta pequeno.

## gin.Context: o parâmetro único do handler

Na stdlib, um handler recebe dois parâmetros separados (`w http.ResponseWriter, r *http.Request`). Em Gin, um handler recebe um único parâmetro, `c *gin.Context`, que embrulha os dois e adiciona funcionalidades extras (acesso a parâmetros de rota, binding de JSON, controle da cadeia de middlewares — ver [Middleware e chain de handlers](../middleware-e-chain-de-handlers/)):

```go
func handler(c *gin.Context) {
    // c.Request  -> o *http.Request original, se você precisar dele diretamente
    // c.Writer   -> o http.ResponseWriter original
    // mas o normal é usar os métodos de conveniência do próprio *gin.Context
}
```

## Path params e query params

```go
r.GET("/pedidos/:id", func(c *gin.Context) {
    id := c.Param("id") // path param — note a sintaxe ":id", diferente do "{id}" da stdlib
    c.JSON(http.StatusOK, gin.H{"id": id})
})

r.GET("/pedidos", func(c *gin.Context) {
    status := c.Query("status")                     // "" se não veio
    pagina := c.DefaultQuery("pagina", "1")          // com valor padrão, se não veio
    c.JSON(http.StatusOK, gin.H{"status": status, "pagina": pagina})
})
```

Repare que Gin usa `:id` para path param (dois pontos), enquanto o `ServeMux` moderno da stdlib usa `{id}` (chaves) — são sintaxes de bibliotecas diferentes para o mesmo conceito, não confundir uma com a outra ao trocar de projeto.

## Corpo da requisição: ShouldBindJSON

```go
type CreateOrderRequest struct {
    ClienteID string  `json:"cliente_id" binding:"required"`
    Valor     float64 `json:"valor" binding:"required,gt=0"`
}

r.POST("/pedidos", func(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // req.ClienteID e req.Valor já vieram decodificados E validados aqui
    c.JSON(http.StatusCreated, gin.H{"cliente_id": req.ClienteID})
})
```

`c.ShouldBindJSON(&req)` faz duas coisas em uma chamada: decodifica o corpo JSON para dentro do struct (igual `json.Unmarshal` faria — ver [JSON e consumo de APIs externas](../json-e-consumo-de-apis-externas/)) **e** valida os campos de acordo com as regras escritas na tag `binding` (`required` = obrigatório, `gt=0` = maior que zero, entre várias outras regras disponíveis). Se qualquer validação falhar, `err` vem preenchido e a requisição nem chega a entrar na lógica de negócio. Na stdlib pura, decodificar e validar são dois passos manuais e separados — Gin junta os dois. O tópico [Validação de input](../validacao-de-input/) aprofunda as regras de `binding` disponíveis.

## Rotas agrupadas com middleware compartilhado

```go
func authMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "token ausente"})
        c.Abort() // interrompe a cadeia — os handlers seguintes não rodam
        return
    }
    c.Next() // deixa a cadeia continuar até o handler final
}

func main() {
    r := gin.Default()

    api := r.Group("/v1")
    api.Use(authMiddleware) // todo handler registrado dentro deste grupo passa por authMiddleware antes
    {
        api.GET("/pedidos", listOrders)
        api.POST("/pedidos", createOrder)
    }

    // rota fora do grupo — não passa por authMiddleware
    r.GET("/status", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    r.Run(":8080")
}
```

`r.Group("/v1")` cria um subconjunto de rotas que compartilham um prefixo de URL (`/v1/pedidos` em vez de `/pedidos`) e podem compartilhar middlewares aplicados só a esse grupo, via `.Use(...)`. As chaves `{ }` ao redor das chamadas dentro do grupo são só um bloco comum do Go, sem significado especial — usadas por convenção de estilo para deixar visualmente claro quais rotas pertencem àquele grupo. [Middleware e chain de handlers](../middleware-e-chain-de-handlers/) detalha `c.Next()`/`c.Abort()` e como a cadeia de middlewares funciona por dentro.

## Escrevendo um middleware próprio, do zero

Um middleware em Gin é só uma função com a mesma assinatura de um handler (`func(c *gin.Context)`) que decide se chama `c.Next()` (deixa passar) ou não. Um exemplo simples, medindo o tempo de cada requisição:

```go
import (
    "log"
    "time"
)

func timingMiddleware(c *gin.Context) {
    inicio := time.Now()
    c.Next() // executa o resto da cadeia (outros middlewares + o handler final)
    duracao := time.Since(inicio)
    log.Printf("%s %s levou %v", c.Request.Method, c.Request.URL.Path, duracao)
}

func main() {
    r := gin.Default()
    r.Use(timingMiddleware) // aplicado a TODAS as rotas, não só a um grupo
    // ... registrar rotas normalmente
    r.Run(":8080")
}
```

`r.Use(...)` no roteador raiz (em vez de num grupo específico) aplica o middleware a todo o servidor.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise gin framework web`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: reescrever a API de tarefas do tópico [net/http e APIs REST](../net-http-e-apis-rest/) usando Gin em vez da stdlib, comparando quantas linhas cada abordagem exige para o mesmo comportamento.
