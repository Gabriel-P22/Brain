# Produção — logging, config, error wrapping

## Log estruturado: o que é e por que importa

"Logar" é imprimir uma mensagem em algum lugar (normalmente o terminal, ou um arquivo) toda vez que algo relevante acontece no programa — um pedido foi criado, uma chamada externa falhou, o servidor subiu. **Log estruturado** é logar não como uma frase solta de texto livre, mas como um conjunto de campos nomeados (chave/valor), geralmente formatados em JSON. A diferença importa em produção porque um log estruturado pode ser buscado e filtrado automaticamente por uma ferramenta ("me mostre todo log com `order_id = 123`"), enquanto uma frase de texto livre só pode ser lida por um humano ou vasculhada com busca de texto simples.

```go
// log de texto livre — funciona pra humano ler, ruim pra ferramenta processar
fmt.Println("pedido 123 criado com valor 42.50")

// log estruturado — mesma informação, mas em campos nomeados e buscáveis
logger.Info("pedido criado", "order_id", "123", "amount", 42.50)
```

## log/slog — logging estruturado é stdlib desde Go 1.21

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("pedido criado", "order_id", order.ID, "amount", order.Amount)
// {"time":"2026-08-08T10:00:00Z","level":"INFO","msg":"pedido criado","order_id":"123","amount":42.5}
```

`slog.New` cria um logger a partir de um **handler** — o handler decide o formato de saída. `slog.NewJSONHandler` produz uma linha de JSON por chamada, ideal pra produção (ferramenta de observabilidade consome JSON facilmente). Existe também `slog.NewTextHandler`, com formato `chave=valor` legível direto no terminal — mais confortável durante desenvolvimento local:

```go
// em dev, formato de texto é mais fácil de ler no olho
devLogger := slog.New(slog.NewTextHandler(os.Stdout, nil))
devLogger.Info("pedido criado", "order_id", order.ID, "amount", order.Amount)
// time=2026-08-08T10:00:00.000-03:00 level=INFO msg="pedido criado" order_id=123 amount=42.5
```

Antes do Go 1.21, log estruturado dependia de uma biblioteca de terceiro (`zap`, `zerolog`, `logrus` são as mais conhecidas do ecossistema) — hoje `log/slog`, já incluído na instalação padrão do Go, cobre o caso comum sem precisar adicionar dependência nenhuma ao projeto.

## Níveis de log

`slog` tem quatro níveis padrão, em ordem crescente de severidade: `Debug`, `Info`, `Warn`, `Error`.

```go
logger.Debug("cache miss", "key", cacheKey)               // detalhe útil só durante investigação
logger.Info("pedido criado", "order_id", order.ID)          // evento normal, esperado
logger.Warn("retry na chamada externa", "tentativa", 2)      // algo incomum, mas o programa se recuperou sozinho
logger.Error("falha ao salvar pedido", "err", err.Error())    // algo quebrou e precisa de atenção
```

Em produção, é comum configurar o handler pra só emitir `Info` pra cima (ignorando `Debug`), reduzindo volume de log sem perder o que realmente importa:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo, // Debug fica silenciado nesse logger
}))
```

## Logger com campos fixos, via `With`

Quando várias chamadas de log dentro do mesmo trecho de código sempre carregam o mesmo campo (por exemplo, o ID de um pedido sendo processado), `logger.With(...)` cria um novo logger que já inclui esses campos em toda chamada seguinte, sem precisar repeti-los:

```go
orderLogger := logger.With("order_id", order.ID)

orderLogger.Info("validando pedido")
orderLogger.Info("pedido salvo no banco")
orderLogger.Error("falha ao notificar cliente", "err", err.Error())
// as três linhas acima já saem com "order_id":"123" incluído automaticamente
```

## Correlation ID por requisição

Um servidor real atende muitas requisições ao mesmo tempo, cada uma rodando concorrentemente (ver [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/)) — os logs de requisições diferentes ficam intercalados no mesmo fluxo de saída. Sem um identificador que amarre "todo log que pertence a essa requisição específica", depurar um problema em produção vira adivinhação. A solução idiomática é gerar um ID único no início de cada requisição e propagá-lo através do `context.Context` (ver [Acesso a dados e context.Context](../acesso-a-dados-e-context/)):

```go
type ctxKey string

const requestIDKey ctxKey = "request_id"

func withRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := uuid.NewString()
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        logger.InfoContext(ctx, "request iniciado", "request_id", id, "path", r.URL.Path)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

`context.WithValue` é o único uso idiomático aceito de "carregar um dado dentro do context" fora de cancelamento/timeout — e mesmo assim só pra metadado transversal como esse ID de correlação, nunca pra dado de negócio (isso sempre deveria ser um parâmetro explícito da função, não algo escondido dentro do `ctx`). Ver [BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#observabilidade) pra por que isso importa tanto em produção.

## Configuração via variável de ambiente

Configuração (endereço do banco, chave de API, porta que o servidor escuta) nunca deveria estar escrita direto no código — isso obrigaria recompilar o programa toda vez que um valor muda entre ambientes (dev, staging, produção), e frequentemente significaria colocar segredo (senha, chave de API) dentro do repositório de código, o que é um risco de segurança sério. O padrão é ler esses valores do ambiente onde o processo está rodando:

```go
type Config struct {
    DatabaseURL string
    Port        string
    LLMAPIKey   string
}

func LoadConfig() (Config, error) {
    key := os.Getenv("LLM_API_KEY")
    if key == "" {
        return Config{}, errors.New("LLM_API_KEY não definida")
    }
    return Config{
        DatabaseURL: os.Getenv("DATABASE_URL"),
        Port:        cmp.Or(os.Getenv("PORT"), "8080"), // cmp.Or (Go 1.22+): primeiro valor não-vazio
        LLMAPIKey:   key,
    }, nil
}
```

`os.Getenv("X")` devolve o valor da variável de ambiente `X`, ou uma string vazia se ela não estiver definida — não existe erro nem exceção aqui, só o zero value de `string`. `cmp.Or` (função da stdlib desde Go 1.22) devolve o primeiro argumento que não é o zero value do seu tipo — aqui, usada pra dar um valor padrão (`"8080"`) quando `PORT` não foi definida.

## Falhar cedo, não na primeira requisição que precisar do valor ausente

```go
// versão ingênua: cada função lê a variável de ambiente na hora que precisa dela
func chamarLLM(prompt string) (string, error) {
    key := os.Getenv("LLM_API_KEY") // e se estiver vazia? só descobre aqui, no meio de uma requisição real
    // ...
}
```

Nessa versão, se `LLM_API_KEY` não estiver definida, o programa sobe normalmente, aceita requisições, e só falha quando a *primeira* requisição que precisa dessa chave chega — o que pode ser minutos ou horas depois do deploy, no pior momento possível (um cliente real recebendo erro). A alternativa correta é centralizar a leitura de toda configuração num único lugar (`LoadConfig`, como no exemplo anterior), chamado uma única vez, no `main`, antes de o servidor começar a aceitar qualquer requisição:

```go
func main() {
    cfg, err := LoadConfig()
    if err != nil {
        log.Fatal(err) // processo nem chega a subir — falha visível, imediata, no boot
    }

    llmClient := anthropic.NewClient(cfg.LLMAPIKey)
    // ... resto do main monta o servidor usando cfg e llmClient já validados
}
```

Isso é "falhar cedo": um erro de configuração vira um crash imediato e óbvio no momento do deploy (fácil de notar, fácil de corrigir), em vez de um bug silencioso que só aparece pra um cliente de produção horas depois. Também tem um benefício de design: `LoadConfig` é o único lugar do programa que chama `os.Getenv` — o resto do código recebe a configuração já validada como parâmetro (`Config`), o que deixa cada função testável sem precisar manipular variável de ambiente global no teste.

Não existe, na stdlib de Go, nenhum carregamento automático de arquivo `.env` — ou a variável já está no ambiente (exportada manualmente, definida no `ENV` de um Dockerfile, injetada por quem orquestra o deploy), ou o próprio código lê um arquivo `.env` explicitamente, usando uma biblioteca pequena como `godotenv`, tipicamente só em desenvolvimento local:

```go
// só em dev: carrega .env pra dentro das variáveis de ambiente do processo, antes de LoadConfig rodar
if os.Getenv("APP_ENV") != "production" {
    _ = godotenv.Load() // ignora erro se o arquivo não existir — em produção não deveria existir mesmo
}
cfg, err := LoadConfig()
```

## Error wrapping, revisitado com contexto de produção

Já visto em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/#wrapping--a-versão-go-do-traceback) — o que muda aqui é o porquê disso importar especificamente em produção. Quando algo falha em produção, a única pista disponível costuma ser a linha de log daquele erro — não dá pra colocar um debugger num servidor rodando ao vivo. A cadeia de `%w` é literalmente o que aparece nessa linha:

```go
func loadUserOrder(ctx context.Context, userID, orderID string) (*Order, error) {
    order, err := repo.FindByID(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("carregando pedido %s do usuário %s: %w", orderID, userID, err)
    }
    return order, nil
}
```

```go
// no handler, ao logar o erro final:
logger.Error("falha na requisição", "err", err.Error())
// se cada camada envolveu o erro com fmt.Errorf(...: %w...), a mensagem final é algo como:
// "carregando pedido 42 do usuário 7: consultando postgres: timeout: context deadline exceeded"
// sem wrapping em nenhuma camada, a mesma linha seria só:
// "context deadline exceeded"
```

A segunda versão (sem wrapping) é tecnicamente o mesmo erro, mas inútil pra quem está lendo o log tentando entender *o que* estava sendo feito quando o timeout aconteceu. Cada `fmt.Errorf("...: %w", err)`, em cada camada por onde o erro passa, adiciona uma frase de contexto — o resultado final é uma trilha legível de "o que estava acontecendo", parecido em espírito com um traceback, mas construído manualmente, camada por camada, por quem escreveu o código.
