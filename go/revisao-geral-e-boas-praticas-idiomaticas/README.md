# Revisão geral e boas práticas idiomáticas

Fechamento do plano de 2 semanas: não introduz conceito novo, consolida os 26 tópicos anteriores (Módulos 1-3) num formato de consulta rápida — recap de armadilhas, checklist idiomático e perguntas típicas de dia-1/entrevista técnica.

## Recap por módulo

- **Módulo 1 (Fundamentos)**: tipagem estática com zero value em vez de `None`, ponteiro explícito, slice compartilhando array subjacente, `error` como valor (não exceção), goroutines/channels, `go.mod`/`go.sum` para dependências.
- **Módulo 2 (SOLID/DDD/Clean Architecture/Clean Code)**: os mesmos princípios que você já usa em Python, aplicados com a gramática de Go — interface pequena descoberta pelo consumidor (não declarada preventivamente), composição via embedding no lugar de herança, camadas separadas por pacote + `internal/` para impedir import de fora do módulo.
- **Módulo 3 (Backend)**: `net/http`/Gin, JSON e struct tags, `context.Context` para cancelamento/timeout, repository pattern, middleware, worker pools/rate limiting, logging estruturado, testes table-driven.

Detalhe de cada um continua só na `README.md` do respectivo tópico — este arquivo não repete explicação, só aponta o que revisar se algo não estiver firme.

## As armadilhas que mais pegam quem vem de Python

Já explicadas no tópico onde surgiram — aqui só a lista de "se esquecer, foi isso":

| Armadilha | Onde foi coberta |
|---|---|
| Slice compartilha array subjacente do original (`append` pode ou não realocar) | [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) |
| Map com zero value `nil` — leitura ok, escrita causa panic | [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) |
| Ignorar `error` (`_ = f()`) é visível no diff, mas ainda assim comum de esquecer | [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) |
| Goroutine leak — goroutine bloqueada pra sempre, sem exceção que avise | [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) |
| Receiver ponteiro vs valor inconsistente no mesmo tipo (mistura gera bug sutil de cópia) | [Structs, métodos e interfaces](../structs-metodos-e-interfaces/) |
| `go.sum` esquecido fora do commit (nunca deveria ser gitignored) | [Gerenciamento de pacotes](../gerenciamento-de-pacotes/) |
| `context.Context` não propagado numa chamada bloqueante — timeout do caller não corta a chamada de baixo | [Acesso a dados e context.Context](../acesso-a-dados-e-context/) |

## Checklist idiomático rápido

Antes de considerar um trecho de Go "pronto", pergunte:

- **Erro tem contexto?** `fmt.Errorf("fazendo X: %w", err)`, nunca erro nu repassado sem dizer onde ocorreu (ver [Clean Code em Go](../clean-code-em-go/)).
- **Interface está do lado de quem consome, não de quem implementa?** — "accept interfaces, return structs". Uma interface com 1 implementação e 0 consumidor externo provavelmente não devia existir ainda (ver [SOLID em Go](../solid-em-go/), Interface Segregation).
- **`gofmt`/`goimports` rodaram?** Não é opinião de estilo, é automático — se o CI reclamar de formatação é porque não rodou.
- **`golangci-lint run` limpo?** Cobre boa parte do que em outra linguagem seria comentário manual de code review (nome não usado, shadow de variável, erro ignorado sem `_ =` explícito, etc).
- **Pacote tem um propósito claro?** Nome de pacote genérico (`utils`, `common`, `helpers`) é sinal de Single Responsibility violado no nível de organização, não só de tipo.
- **Concorrência tem forma de terminar?** Toda goroutine disparada precisa de um caminho claro de retorno — `context` cancelável, `WaitGroup`, ou channel fechado. "Disparei e nunca mais pensei nisso" é o padrão do goroutine leak.

## Perguntas frequentes de dia-1 / entrevista técnica

**Por que Go não tem `try`/`except`?**
Erro como valor de retorno força o chamador a decidir explicitamente o que fazer — não existe caminho de erro "silencioso" subindo pilha acima sem ser declarado na assinatura da função. `panic`/`recover` existe, mas é para estado realmente inconsistente, não fluxo de erro esperado.

**Qual a diferença entre array e slice?**
Array tem tamanho fixo no tipo (`[5]int` é um tipo diferente de `[10]int`). Slice é a estrutura dinâmica de verdade (ponteiro + tamanho + capacidade) sobre um array subjacente — quase todo código Go usa slice, array puro é raro no dia a dia.

**Quando usar goroutine vs quando não usar?**
Quando há trabalho que pode rodar de forma independente e o custo de coordenação (channel, `sync`) compensa o ganho de paralelismo/concorrência. Para trabalho puramente sequencial e rápido, disparar goroutine só adiciona overhead de agendamento e complexidade sem ganho.

**O que `context.Context` resolve que um simples timeout não resolve?**
Propagação: um `context` carrega cancelamento/deadline através de toda uma cadeia de chamadas (HTTP handler → service → repository → driver de banco), permitindo que cortar no topo corte automaticamente as chamadas bloqueantes de baixo — um timeout isolado numa função só não se propaga pra quem ela chama.

**Por que Go não tem herança?**
Decisão deliberada de design: composição (embedding de struct/interface) resolve reuso de código sem a fragilidade do "diamond problem" ou hierarquias profundas — mesmo motivo pelo qual "favoreça composição sobre herança" já é um princípio recomendado em Python/OO em geral, só que em Go é a única opção, não uma escolha de estilo.

**O que `go.sum` garante que `go.mod` sozinho não garante?**
Integridade: `go.mod` diz qual versão; `go.sum` trava o hash de cada dependência (direta e transitiva) baixada, então um build não pode silenciosamente puxar um artefato alterado com o mesmo número de versão.

## Ferramentas — só o pointer, sem re-explicar

`gofmt`, `goimports` e `golangci-lint` já foram cobertos como convenção automática em [Clean Code em Go](../clean-code-em-go/) e no exemplo idiomático em [go/config/reference/CLEAN-CODE.md](../config/reference/CLEAN-CODE.md). Rotina prática antes de qualquer commit:

```
gofmt -l .          # lista arquivo fora do padrão (vazio = tudo formatado)
go vet ./...        # erros comuns que compilam mas são suspeitos (printf mal formatado, etc)
golangci-lint run    # agrega múltiplos linters (inclui vet, staticcheck, e outros)
go test ./...        # bateria de testes
```

## Gaps conhecidos — o que este plano não cobriu a fundo

Honestidade sobre o que ficou de fora das 2 semanas (não é obrigatório pra dia-1, mas vale saber que existe):

- **Generics** (`[T any]`, desde Go 1.18) — não apareceu em nenhum tópico. Útil saber que existe e a sintaxe básica, mas a maior parte de código de backend do dia a dia ainda é escrita sem generics.
- **Reflection** (`reflect`) — usado internamente por libs (JSON, ORMs), raramente escrito à mão.
- **cgo / interop com C** — irrelevante pro perfil de API/infra do cargo.
- **Build tags e cross-compilation avançada** — `GOOS`/`GOARCH` foram mencionados de passagem, sem aprofundar em matriz de build multi-plataforma.

Se alguma dessas aparecer em entrevista, é razoável responder "sei que existe e pra que serve, não usei na prática ainda" — resposta honesta de alguém que fez 2 semanas de curva de aprendizado, não uma lacuna pra esconder.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise revisão geral`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Se quiser um exercício de fechamento de verdade, o candidato natural é revisitar o projeto proposto em [Revisão e projeto CLI](../revisao-e-projeto-cli/) ou o esboço de [Projeto final — API + LLM](../projeto-final-api-llm/), já que nenhum dos dois foi implementado ainda.
