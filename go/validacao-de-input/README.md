# Validação de input

## O que é "validar input" e por que isso não é regra de negócio

Quando uma API recebe um pedido (um `POST /orders` com um corpo JSON, por exemplo), antes de fazer qualquer coisa com esses dados é preciso checar se eles fazem sentido no formato básico: o campo obrigatório veio preenchido? O número é positivo? O texto de e-mail tem cara de e-mail? Isso é **validação de input** — checagem de *formato*, não de *regra de negócio*.

A diferença importa porque são responsabilidades de camadas diferentes (ver [Clean Architecture](../clean-architecture-separacao-em-camadas/)):

- **Validação de input** (aqui) — "esse valor tem o formato esperado?" Mora na borda da aplicação, antes do dado sequer chegar na regra de negócio.
- **Regra de negócio** — "esse valor, mesmo bem formatado, é permitido *neste momento*?" Um exemplo: `Order.Cancel()` recusando cancelar um pedido que já foi enviado — o campo está perfeitamente bem formatado, o problema é o estado do sistema. Isso mora no domínio, não na borda.

## Validação manual, com um método `Validate`

A forma mais simples de validar em Go, sem nenhuma biblioteca externa, é escrever um método comum que checa cada campo e devolve um `error` se algo estiver errado:

```go
type CreateOrderRequest struct {
    Amount float64 `json:"amount"`
    Email  string  `json:"email"`
}

func (r CreateOrderRequest) Validate() error {
    if r.Amount <= 0 {
        return errors.New("amount deve ser positivo")
    }
    if !strings.Contains(r.Email, "@") {
        return errors.New("email inválido")
    }
    return nil
}
```

Não existe nenhuma anotação especial nem nenhum mecanismo "mágico" rodando por trás — `Validate()` é um método comum, como qualquer outro, que só é chamado porque alguém decide chamá-lo explicitamente no handler:

```go
func createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "corpo inválido", http.StatusBadRequest)
        return
    }
    if err := req.Validate(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    // a partir daqui, req já está validado — o resto do handler pode confiar nos dados
}
```

## Ingênuo vs extraído: por que a validação não deveria morar solta dentro do handler

Uma primeira versão, sem separar a validação num método próprio, jogaria toda a checagem direto dentro da função do handler:

```go
// versão ingênua: validação misturada com o resto da lógica do handler
func createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "corpo inválido", http.StatusBadRequest)
        return
    }

    if req.Amount <= 0 {
        http.Error(w, "amount deve ser positivo", http.StatusBadRequest)
        return
    }
    if !strings.Contains(req.Email, "@") {
        http.Error(w, "email inválido", http.StatusBadRequest)
        return
    }

    // só agora começa a lógica de verdade: criar o pedido, salvar, etc.
    order, err := service.Create(r.Context(), req.Amount, req.Email)
    // ...
}
```

Funciona, mas mistura duas responsabilidades diferentes dentro da mesma função: "o formato do pedido está correto?" e "o que fazer com um pedido correto?". Isso viola o Single Responsibility Principle (ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md#s--single-responsibility-principle)) — o handler passa a ter dois motivos pra mudar (mudou uma regra de formato, ou mudou o fluxo de negócio). Extrair `Validate()` como método do próprio tipo `CreateOrderRequest` resolve isso: o handler fica só com "decodificar, validar, delegar pro serviço", e a regra de formato de cada campo fica testável isoladamente, sem precisar simular uma requisição HTTP inteira pra testar se `Amount <= 0` é rejeitado.

## Validação declarativa via struct tag (ecossistema Gin)

Quando o projeto usa o framework Gin (ver [Gin (framework web)](../gin-framework-web/)), existe uma forma declarativa de descrever a validação direto na definição do `struct`, usando a mesma sintaxe de struct tag já vista em [JSON e consumo de APIs externas](../json-e-consumo-de-apis-externas/):

```go
type CreateOrderRequest struct {
    Amount float64 `json:"amount" binding:"required,gt=0"`
    Email  string  `json:"email" binding:"required,email"`
}

// no handler Gin:
func createOrder(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // req já chega validado até aqui
}
```

`binding:"required,gt=0"` lê-se como uma lista de regras separadas por vírgula: `required` (o campo não pode ficar no zero value) e `gt=0` (maior que zero). `c.ShouldBindJSON` decodifica o JSON *e* roda essas regras num passo só — por trás, isso é a biblioteca `go-playground/validator`, o pacote mais usado do ecossistema Go pra esse tipo de validação declarativa. Ainda assim, é mais manual do que parece à primeira vista: diferente de um método `Validate()` escrito à mão, essa abordagem não gera documentação de API sozinha, e as mensagens de erro que ela produz por padrão tendem a ser técnicas (nome do campo em inglês, nome da regra violada) — muitas equipes escrevem uma camada fina por cima pra traduzir isso em mensagens mais amigáveis pro cliente final.

## Coletando todos os erros, não só o primeiro

Os exemplos acima param no primeiro campo inválido que encontram (`Validate()` retorna assim que acha o primeiro problema). Em produção, geralmente é melhor devolver *todos* os problemas de uma vez, pra quem está chamando a API não precisar corrigir um campo, reenviar, descobrir o próximo erro, corrigir de novo:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (r CreateOrderRequest) Validate() []ValidationError {
    var errs []ValidationError

    if r.Amount <= 0 {
        errs = append(errs, ValidationError{Field: "amount", Message: "deve ser positivo"})
    }
    if !strings.Contains(r.Email, "@") {
        errs = append(errs, ValidationError{Field: "email", Message: "formato inválido"})
    }

    return errs // slice vazio (não nil-erro) se tudo estiver certo
}

// no handler:
if errs := req.Validate(); len(errs) > 0 {
    w.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(w).Encode(map[string]any{"errors": errs})
    return
}
```

Aqui `Validate()` não devolve mais um único `error` — devolve uma lista (`[]ValidationError`) de todos os campos que falharam, cada um com seu próprio nome de campo e mensagem. É um pouco mais de código, mas evita a experiência frustrante de um cliente de API corrigindo erro "um de cada vez".

## Erro de validação é 400, sempre

Ver [Tratamento de erros e exceções em APIs](../tratamento-de-erros-e-excecoes-em-apis/) — erro de validação nunca deveria virar `500`. A diferença central: se a mensagem de erro pode ir direto pro cliente (como nos exemplos acima), ela é informação útil pra ele corrigir o pedido — bem diferente de um erro de infra, cuja mensagem detalhada deve ficar só no log do servidor (ver [Produção — logging, config, error wrapping](../producao-logging-config-error-wrapping/)).
