# JSON e consumo de APIs externas

## O que é JSON, do zero

**JSON** (JavaScript Object Notation) é um formato de texto para representar dados estruturados, organizado em pares de chave e valor. Apesar do nome vir de JavaScript, JSON hoje é o formato padrão de dado trocado entre sistemas na internet, usado por praticamente qualquer linguagem de programação — inclusive Go.

```json
{
  "id": "42",
  "nome": "Mesa de escritório",
  "preco": 350.0,
  "disponivel": true,
  "tags": ["móveis", "escritório"],
  "fornecedor": {
    "id": "7",
    "nome": "Marcenaria Silva"
  }
}
```

Os tipos que existem em JSON são poucos e simples: texto (entre aspas), número, `true`/`false`, `null` (ausência de valor), lista (entre colchetes `[ ]`) e objeto (entre chaves `{ }`, um conjunto de pares chave-valor, como no exemplo acima). Qualquer estrutura de dado mais complexa é montada combinando esses seis tipos básicos — um objeto dentro de outro, uma lista de objetos, etc.

Uma API REST (ver [net/http e APIs REST](../net-http-e-apis-rest/)) tipicamente manda e recebe JSON no corpo das requisições HTTP — é por isso que "trabalhar com API" em Go, na prática do dia a dia, é em grande parte "converter entre struct Go e JSON".

## O pacote encoding/json

O pacote da biblioteca padrão que faz essa conversão nos dois sentidos se chama `encoding/json`. Ele resolve dois problemas complementares:

- **Serializar** (transformar um valor Go em texto JSON) — a função principal é `json.Marshal`.
- **Desserializar** (transformar texto JSON em um valor Go) — a função principal é `json.Unmarshal`.

```go
import "encoding/json"

type Produto struct {
    ID     string
    Nome   string
    Preco  float64
}

p := Produto{ID: "42", Nome: "Mesa", Preco: 350.0}

dados, err := json.Marshal(p)
if err != nil {
    // Marshal só falha em casos raros (ex: tentar serializar um tipo que o
    // encoding/json não sabe representar, como uma função ou um channel)
    panic(err)
}
fmt.Println(string(dados)) // {"ID":"42","Nome":"Mesa","Preco":350}
```

Repare que os nomes dos campos no JSON de saída ficaram exatamente como estão no struct Go (`ID`, `Nome`, `Preco`, em maiúscula) — isso porque o `encoding/json` usa o nome do campo Go como nome padrão da chave JSON, quando você não diz nada diferente. Na próxima seção você vê como controlar esse nome.

## Struct tags: metadados anexados a um campo

Uma **struct tag** é um texto especial que você pode anexar a um campo de struct, entre crases, logo depois do tipo do campo. Ela não muda o comportamento do campo em si — é só um pedaço de metadado que bibliotecas (como `encoding/json`) sabem ler e interpretar:

```go
type Order struct {
    ID     string  `json:"id"`
    Amount float64 `json:"amount"`
    Note   string  `json:"note,omitempty"` // omite esta chave do JSON se o valor for zero value
    secret string  `json:"-"`              // não exportado: nunca serializa, mesmo sem essa tag
}
```

Cada parte dessa struct tag tem um papel:

- `` `json:"id"` `` — diz ao `encoding/json` para usar `"id"` (minúsculo) como nome da chave no JSON, em vez do nome padrão `"ID"`. Isso é necessário porque convenções de nomenclatura em JSON costumam usar `snake_case` ou `camelCase`, diferente da convenção `PascalCase` que Go usa para campos exportados.
- `` `json:"note,omitempty"` `` — a opção `omitempty` (separada por vírgula, depois do nome) diz para **não incluir essa chave no JSON de saída** se o valor do campo for o "zero value" do seu tipo (string vazia, `0`, `false`, slice/map/ponteiro `nil` — ver [Sintaxe básica](../sintaxe-basica/) para a explicação de zero value). É a forma mais próxima que existe em Go de "campo opcional" — não existe um tipo genérico embutido de "valor que pode ou não estar presente" além disso e de ponteiros.
- `` `json:"-"` `` — o traço sozinho diz "nunca serialize este campo, ponto final", mesmo que ele fosse exportado.

Um detalhe crítico: `encoding/json` **só consegue ler ou escrever campos exportados** (que começam com letra maiúscula — ver a explicação de visibilidade em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)). O campo `secret string` no exemplo acima já não seria serializado mesmo sem a tag `json:"-"`, porque começa com letra minúscula — a tag ali é redundante, mas deixada por clareza para quem lê o código. A tag controla o **nome/comportamento** de um campo já exportado; ela nunca torna um campo não-exportado visível ao JSON.

## Marshal/Unmarshal: exemplo completo nos dois sentidos

```go
type Produto struct {
    ID         string   `json:"id"`
    Nome       string   `json:"nome"`
    Preco      float64  `json:"preco"`
    Disponivel bool     `json:"disponivel"`
    Tags       []string `json:"tags,omitempty"`
}

// struct -> JSON
p := Produto{ID: "42", Nome: "Mesa", Preco: 350, Disponivel: true, Tags: []string{"móveis"}}
dados, err := json.Marshal(p)
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(dados))
// {"id":"42","nome":"Mesa","preco":350,"disponivel":true,"tags":["móveis"]}

// JSON -> struct
entrada := []byte(`{"id":"7","nome":"Cadeira","preco":220.5,"disponivel":false}`)
var outro Produto
if err := json.Unmarshal(entrada, &outro); err != nil {
    log.Fatal(err)
}
fmt.Printf("%+v\n", outro)
// {ID:7 Nome:Cadeira Preco:220.5 Disponivel:false Tags:[]}
```

Dois pontos que costumam surpreender:

- **Sem exceção em erro de parse** — se o texto de entrada não é um JSON válido (ex: faltou uma chave de fechamento), `Unmarshal` não interrompe o programa; ele devolve um `error` comum, como qualquer função Go que pode falhar (ver [O que é Go](../o-que-e-go/) sobre erro como valor). Você trata isso com o mesmo `if err != nil` de sempre.
- **`Unmarshal` recebe um ponteiro** (`&outro`, não `outro`) — porque `Unmarshal` precisa **modificar** a struct que você passou, preenchendo os campos com o que veio do JSON. Passar `outro` (sem `&`) copiaria a struct, e o `Unmarshal` preencheria a cópia — a variável original ficaria vazia e nenhum erro apareceria, porque sintaticamente isso compila (`Unmarshal` aceita `interface{}`, o que passa a checagem do compilador mesmo passando o tipo errado por engano). Esse é um erro clássico de quem está começando com `encoding/json`.
- **Campo desconhecido no JSON de entrada é ignorado silenciosamente por padrão.** Se `entrada` tivesse uma chave `"cor": "azul"` que não existe no struct `Produto`, o `Unmarshal` simplesmente ignora essa chave, sem erro nenhum. Se você quer rejeitar JSON com campos que a sua struct não conhece (útil para pegar erros de digitação ou de contrato de API cedo), use um `json.Decoder` com `DisallowUnknownFields()`:

```go
decoder := json.NewDecoder(strings.NewReader(string(entrada)))
decoder.DisallowUnknownFields()
var outro Produto
if err := decoder.Decode(&outro); err != nil {
    // agora um campo desconhecido no JSON vira erro
    log.Fatal(err)
}
```

## Marshal/Unmarshal vs Encode/Decode: qual usar

`encoding/json` tem duas famílias de função que fazem, essencialmente, a mesma coisa, mas com uma diferença de onde o dado vive:

- `json.Marshal`/`json.Unmarshal` trabalham com `[]byte` (um bloco de bytes já inteiro na memória) — bons quando você já tem o JSON inteiro como uma variável, ou precisa do JSON inteiro como uma variável antes de usá-lo.
- `json.NewEncoder(w).Encode(...)`/`json.NewDecoder(r).Decode(...)` trabalham direto com um `io.Writer`/`io.Reader` — um **fluxo** de bytes, sem precisar montar o `[]byte` inteiro na memória primeiro. É a forma idiomática quando você já está lidando com um `io.Writer`/`io.Reader`, como o corpo de uma requisição/resposta HTTP (`http.ResponseWriter` e `r.Body` são justamente isso — foi o que apareceu nos exemplos de [net/http e APIs REST](../net-http-e-apis-rest/)).

```go
// Encode: escreve o JSON direto no corpo da resposta HTTP, sem passar por []byte no meio
json.NewEncoder(w).Encode(produto)

// Decode: lê o JSON direto do corpo da requisição HTTP recebida
json.NewDecoder(r.Body).Decode(&produto)
```

Para APIs HTTP, prefira `Encoder`/`Decoder` — evita alocar um `[]byte` inteiro do corpo na memória só para depois descartá-lo.

## Consumindo uma API externa

Quando o seu programa Go é quem faz o papel de **cliente** — chamando uma API de outro sistema, em vez de expor a própria — o fluxo típico é: montar a requisição, enviá-la, checar o status da resposta, decodificar o corpo:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

// PrevisaoTempo representa o formato de resposta de uma API externa fictícia de clima.
type PrevisaoTempo struct {
    Cidade      string  `json:"cidade"`
    TemperaturaC float64 `json:"temperatura_c"`
}

func buscarPrevisao(ctx context.Context, cidade string) (*PrevisaoTempo, error) {
    url := fmt.Sprintf("https://api.exemplo.com/clima?cidade=%s", cidade)

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("montando requisição: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("chamando API externa: %w", err)
    }
    defer resp.Body.Close() // sempre fechar o corpo, mesmo em caso de erro depois disso

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("API externa devolveu status %d", resp.StatusCode)
    }

    var previsao PrevisaoTempo
    if err := json.NewDecoder(resp.Body).Decode(&previsao); err != nil {
        return nil, fmt.Errorf("decodificando resposta: %w", err)
    }
    return &previsao, nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    previsao, err := buscarPrevisao(ctx, "Sao-Paulo")
    if err != nil {
        fmt.Println("erro:", err)
        return
    }
    fmt.Printf("%s: %.1f°C\n", previsao.Cidade, previsao.TemperaturaC)
}
```

Alguns pontos que valem destacar:

- `context.WithTimeout(context.Background(), 5*time.Second)` cria um **contexto** com um prazo máximo de 5 segundos — se a API externa não responder a tempo, a requisição é cancelada automaticamente, em vez de travar o programa esperando para sempre. O tópico [Acesso a dados e context.Context](../acesso-a-dados-e-context/) aprofunda o que é `context.Context` e por que ele se propaga por toda chamada de I/O em Go idiomático.
- `defer resp.Body.Close()` é obrigatório em toda resposta HTTP bem-sucedida — esquecer isso vaza a conexão de rede subjacente (ela fica presa, não é devolvida para reuso), e com o tempo isso esgota o número de conexões disponíveis. `defer` aqui funciona como uma garantia de limpeza que roda mesmo que uma linha depois dela dê erro e a função retorne cedo (ver `defer` em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)).
- Checar `resp.StatusCode` manualmente é necessário — diferente de uma falha de rede (`err != nil` em `http.DefaultClient.Do`), um servidor que responde com `404` ou `500` **não** produz um `error` do Go automaticamente. A requisição "funcionou" do ponto de vista de rede; o conteúdo da resposta é que representa uma falha lógica, e cabe ao seu código checar isso.

## Enviando dados para uma API externa (POST)

O mesmo padrão, mas agora serializando uma struct Go para JSON antes de mandar, no corpo da requisição:

```go
type NovoPedido struct {
    ClienteID string  `json:"cliente_id"`
    Valor     float64 `json:"valor"`
}

func criarPedidoExterno(ctx context.Context, pedido NovoPedido) error {
    corpo, err := json.Marshal(pedido)
    if err != nil {
        return fmt.Errorf("serializando pedido: %w", err)
    }

    req, err := http.NewRequestWithContext(
        ctx, http.MethodPost, "https://api.exemplo.com/pedidos", bytes.NewReader(corpo),
    )
    if err != nil {
        return fmt.Errorf("montando requisição: %w", err)
    }
    req.Header.Set("Content-Type", "application/json") // avisa o servidor do formato do corpo

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return fmt.Errorf("chamando API externa: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        return fmt.Errorf("API externa devolveu status %d", resp.StatusCode)
    }
    return nil
}
```

`bytes.NewReader(corpo)` transforma o `[]byte` que veio do `json.Marshal` num `io.Reader`, porque é isso que `http.NewRequestWithContext` espera como corpo da requisição. `req.Header.Set("Content-Type", "application/json")` é uma boa prática: avisa explicitamente ao servidor do outro lado como interpretar os bytes do corpo.

Esse é exatamente o mecanismo que o tópico [Projeto final — API + LLM](../projeto-final-api-llm/) usa para chamar a API de um provedor de modelo de linguagem a partir de um serviço Go.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise json e consumo de apis externas`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: escrever uma função que consome uma API pública gratuita real (ex: uma API de cotação de moeda ou de dados de CEP), decodifica a resposta numa struct Go, e trata tanto erro de rede quanto status code diferente de `200`.
