# GO-MODULES

Conteúdo por tópico de Go — acumula contexto (notas, dúvidas, erros recorrentes) conforme o tópico é estudado, independente de quando isso acontece. O cronograma (quando estudar o quê) fica em [GO_STUDY_PLAN.md](GO_STUDY_PLAN.md); aqui fica o "o quê" de cada tópico.

Padrão do vault: cada assunto grande ganha seu próprio `*-MODULES.md` (este é o de Go; python, arquitetura e IA seguem o mesmo formato quando chegar a vez).

Ordem pensada pro cargo de backend: fundamentos → boas práticas de engenharia (pra já ter o vocabulário de camadas/design antes de aplicar em backend real) → backend em Go → revisão final.

---

## Módulo 1: Fundamentos

### Tópico: O que é Go
- Status: [x]
- Dia do plano: 1
- Conteúdo: origem e filosofia da linguagem, diferenças em relação a Python (compilada/estática, concorrência nativa, sem herança, erro como valor, deploy em binário único), como instalar, diferenciais (binário estático, compilação rápida, concorrência, tooling embutido, stdlib de rede) e contrapartidas honestas (menos expressiva pra prototipagem, ecossistema menor em ML/data).
- Contexto/notas: Coberto — README em `go/o-que-e-go/`.

### Tópico: Sintaxe básica
- Status: [x]
- Dia do plano: 1
- Conteúdo: setup (toolchain, `go mod`), variáveis, tipos (numéricos com largura explícita, string, bool, byte, rune), zero value, conversão explícita, funções como tipo de primeira classe, controle de fluxo (`if`, `for` — Go só tem `for`), múltiplos retornos.
- Contexto/notas: Coberto — README em `go/sintaxe-basica/`. Exercício rascunho em `exercise/` (soma + divide com erro).

### Tópico: Structs, métodos e interfaces
- Status: [x]
- Dia do plano: 2
- Conteúdo: structs e métodos (substituem classes), interfaces (tipagem estrutural implícita — bem diferente de Python/ABC).
- Contexto/notas: Coberto — README em `go/structs-metodos-e-interfaces/`: receiver valor vs ponteiro, embedding (composição, não herança), satisfação implícita de interface, zero value struct utilizável.

### Tópico: Ponteiros, slices e maps
- Status: [x]
- Dia do plano: 3
- Conteúdo: ponteiros (novidade real vs Python), arrays vs slices, maps.
- Contexto/notas: Coberto — README em `go/ponteiros-slices-e-maps/`: `&`/`*`, slice compartilhando array subjacente (gotcha), map com zero value nil (panic ao escrever) e sem ordem de iteração garantida.

### Tópico: Tratamento de erros e pacotes
- Status: [x]
- Dia do plano: 4
- Conteúdo: `error` como valor (sem exceptions), organização de pacotes/módulos.
- Contexto/notas: Coberto — README em `go/tratamento-de-erros-e-pacotes/`: error como interface, erro customizado, wrapping com `%w` + `errors.Is`/`errors.As`, panic/recover não é try/except, visibilidade por caixa da letra.

### Tópico: Concorrência — goroutines e channels
- Status: [x]
- Dia do plano: 5
- Conteúdo: goroutines, channels — o maior diferencial de Go, crítico pro papel de infra.
- Contexto/notas: Coberto — README em `go/concorrencia-goroutines-e-channels/`: `go`, channel bufferizado/não, close+range, goroutine leak, closure de loop var (fixado desde Go 1.22).

### Tópico: Concorrência — sync, select e testes
- Status: [x]
- Dia do plano: 6
- Conteúdo: `sync` (WaitGroup, Mutex), `select`, `testing` (table-driven tests).
- Contexto/notas: Coberto — README em `go/concorrencia-sync-select-e-testes/`: WaitGroup+defer, Mutex+race detector, select com time.After, table-driven test com t.Run.

### Tópico: Revisão e projeto CLI
- Status: [x]
- Dia do plano: 7
- Conteúdo: consolidação da semana 1 num CLI pequeno.
- Contexto/notas: Coberto — README em `go/revisao-e-projeto-cli/`: recap do módulo + proposta de projeto (verificador de URLs concorrente via `flag`). Projeto em si ainda não implementado, sem exercício gerado.

## Módulo 2: Boas práticas de engenharia

### Tópico: SOLID em Go
- Status: [ ]
- Dia do plano: —
- Conteúdo: os 5 princípios aplicados via interfaces pequenas/implícitas, composição.
- Contexto/notas: —

### Tópico: DDD em Go
- Status: [ ]
- Dia do plano: —
- Conteúdo: entidades, agregados, repositórios, bounded contexts, como isso vira packages em Go.
- Contexto/notas: —

### Tópico: Clean Architecture / separação em camadas
- Status: [ ]
- Dia do plano: —
- Conteúdo: fronteiras de camada, injeção de dependência sem framework.
- Contexto/notas: —

## Módulo 3: Backend em Go

### Tópico: net/http e APIs REST
- Status: [ ]
- Dia do plano: 8
- Conteúdo: `net/http`, API REST via stdlib.
- Contexto/notas: —

### Tópico: Gin (framework web)
- Status: [ ]
- Dia do plano: —
- Conteúdo: roteamento, grupos de rota, binding de request, comparação com o que dá pra fazer só com stdlib.
- Contexto/notas: —

### Tópico: JSON e consumo de APIs externas
- Status: [ ]
- Dia do plano: 9
- Conteúdo: JSON, struct tags, consumir APIs externas (base pra integrar com LLMs a partir do Go).
- Contexto/notas: —

### Tópico: Acesso a dados e context.Context
- Status: [ ]
- Dia do plano: 10
- Conteúdo: `database/sql`/driver, `context.Context` (cancelamento, timeouts).
- Contexto/notas: —

### Tópico: Migrations de banco de dados
- Status: [ ]
- Dia do plano: —
- Conteúdo: ferramentas de migration em Go (ex: golang-migrate/goose), versionamento de schema.
- Contexto/notas: —

### Tópico: Repository pattern na prática
- Status: [ ]
- Dia do plano: —
- Conteúdo: interface de repositório no domínio, implementação concreta na infra — amarra direto com Clean Architecture do Módulo 2.
- Contexto/notas: —

### Tópico: Tratamento de erros e exceções em APIs
- Status: [ ]
- Dia do plano: —
- Conteúdo: tipos de erro customizados, handler central de erro, mapear erro de domínio para status HTTP.
- Contexto/notas: —

### Tópico: Middleware e chain de handlers
- Status: [ ]
- Dia do plano: —
- Conteúdo: auth, logging, recovery de panic, composição de middlewares.
- Contexto/notas: —

### Tópico: Validação de input
- Status: [ ]
- Dia do plano: —
- Conteúdo: struct tags de validação, validar request antes de chegar na regra de negócio.
- Contexto/notas: —

### Tópico: Padrões de concorrência para infra
- Status: [ ]
- Dia do plano: 11
- Conteúdo: worker pools, rate limiting, pipelines.
- Contexto/notas: —

### Tópico: Produção — logging, config, error wrapping
- Status: [ ]
- Dia do plano: 12
- Conteúdo: logging estruturado, config/env vars, `fmt.Errorf` + `%w`.
- Contexto/notas: —

### Tópico: Testes (unitários e integração)
- Status: [ ]
- Dia do plano: —
- Conteúdo: table-driven tests aplicados a handlers/repositórios, mocks via interface, testes de integração com banco real/testcontainers.
- Contexto/notas: —

### Tópico: Projeto final — API + LLM
- Status: [ ]
- Dia do plano: 13
- Conteúdo: serviço Go que expõe API e chama uma API de LLM (API + infra + IA juntos).
- Contexto/notas: —

## Módulo 4: Revisão final

### Tópico: Revisão geral e boas práticas idiomáticas
- Status: [ ]
- Dia do plano: 14
- Conteúdo: gaps, perguntas de entrevista/dia-1, `gofmt`/`golangci-lint`, idiomatic Go.
- Contexto/notas: —
