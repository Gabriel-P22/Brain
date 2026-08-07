# GO-MODULES

Conteúdo por tópico de Go — acumula contexto (notas, dúvidas, erros recorrentes) conforme o tópico é estudado, independente de quando isso acontece. O cronograma (quando estudar o quê) fica em [GO_STUDY_PLAN.md](GO_STUDY_PLAN.md); aqui fica o "o quê" de cada tópico.

Padrão do vault: cada assunto grande ganha seu próprio `*-MODULES.md` (este é o de Go; python, arquitetura e IA seguem o mesmo formato quando chegar a vez).

---

## Módulo 1: Fundamentos

### Tópico: Sintaxe básica
- Status: [ ]
- Dia do plano: 1
- Conteúdo: setup (toolchain, `go mod`), variáveis, tipos, `const`, controle de fluxo (`if`, `for` — Go só tem `for`), funções, múltiplos retornos.
- Contexto/notas: —

### Tópico: Structs, métodos e interfaces
- Status: [ ]
- Dia do plano: 2
- Conteúdo: structs e métodos (substituem classes), interfaces (tipagem estrutural implícita — bem diferente de Python/ABC).
- Contexto/notas: —

### Tópico: Ponteiros, slices e maps
- Status: [ ]
- Dia do plano: 3
- Conteúdo: ponteiros (novidade real vs Python), arrays vs slices, maps.
- Contexto/notas: —

### Tópico: Tratamento de erros e pacotes
- Status: [ ]
- Dia do plano: 4
- Conteúdo: `error` como valor (sem exceptions), organização de pacotes/módulos.
- Contexto/notas: —

### Tópico: Concorrência — goroutines e channels
- Status: [ ]
- Dia do plano: 5
- Conteúdo: goroutines, channels — o maior diferencial de Go, crítico pro papel de infra.
- Contexto/notas: —

### Tópico: Concorrência — sync, select e testes
- Status: [ ]
- Dia do plano: 6
- Conteúdo: `sync` (WaitGroup, Mutex), `select`, `testing` (table-driven tests).
- Contexto/notas: —

### Tópico: Revisão e projeto CLI
- Status: [ ]
- Dia do plano: 7
- Conteúdo: consolidação da semana 1 num CLI pequeno.
- Contexto/notas: —

## Módulo 2: Aplicado ao trabalho

### Tópico: net/http e APIs REST
- Status: [ ]
- Dia do plano: 8
- Conteúdo: `net/http`, API REST via stdlib e depois framework leve (chi/gin).
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

### Tópico: Projeto final — API + LLM
- Status: [ ]
- Dia do plano: 13
- Conteúdo: serviço Go que expõe API e chama uma API de LLM (API + infra + IA juntos).
- Contexto/notas: —

### Tópico: Revisão geral e boas práticas idiomáticas
- Status: [ ]
- Dia do plano: 14
- Conteúdo: gaps, perguntas de entrevista/dia-1, `gofmt`/`golangci-lint`, idiomatic Go.
- Contexto/notas: —

## Módulo 3: Engenharia de software (transversal)

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
