# JSON e consumo de APIs externas

## Struct tags — o `json.dumps`/`pydantic` do Go

```go
type Order struct {
    ID     string  `json:"id"`
    Amount float64 `json:"amount"`
    Note   string  `json:"note,omitempty"`   // omite do JSON se for zero value
    secret string  `json:"-"`                 // não exportado: nunca serializa mesmo sem a tag
}
```

`encoding/json` só serializa campo exportado (`PascalCase`) — struct tag controla o nome/comportamento no JSON, não a visibilidade em Go (isso já é a caixa da letra). `omitempty` é o mais próximo que existe de "campo opcional" — não existe `Optional[T]` do Python aqui, é convenção de zero value + tag.

## Marshal/Unmarshal

```go
data, err := json.Marshal(order)          // struct -> JSON
err = json.Unmarshal(data, &order)          // JSON -> struct (nota o ponteiro)
```

Sem exception em erro de parse — `err` como sempre. Campo desconhecido no JSON de entrada é ignorado silenciosamente por padrão (diferente de um `pydantic` estrito, que rejeita campo extra por padrão).

## Consumindo API externa

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := http.DefaultClient.Do(req)
if err != nil {
    return nil, fmt.Errorf("chamando API externa: %w", err)
}
defer resp.Body.Close()

var result Response
if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
    return nil, fmt.Errorf("decodificando resposta: %w", err)
}
```

`context.WithTimeout` amarrado na request é o padrão idiomático pra nunca deixar uma chamada de rede travar pra sempre — ver mais sobre `context.Context` em [Acesso a dados e context.Context](../acesso-a-dados-e-context/). `defer resp.Body.Close()` é obrigatório — esquecer isso vaza conexão (mais um caso do `defer` como "finally" implícito, já visto no Módulo 1).

Esse é exatamente o mecanismo que o [Projeto final — API + LLM](../projeto-final-api-llm/) usa pra chamar a API de um provedor de LLM a partir de Go.
