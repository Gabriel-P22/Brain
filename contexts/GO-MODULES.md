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
- Status: [x]
- Dia do plano: —
- Conteúdo: os 5 princípios aplicados via interfaces pequenas/implícitas, composição.
- Contexto/notas: Coberto — README em `go/solid-em-go/`: síntese (não repete contexts/common/SOLID.md nem go/config/reference/SOLID.md), como os 5 se combinam num pacote pequeno, erros comuns vindo de Python (embedding como herança falsa, interface grande prematura, DI framework desnecessário).

### Tópico: DDD em Go
- Status: [x]
- Dia do plano: —
- Conteúdo: entidades, agregados, repositórios, bounded contexts, como isso vira packages em Go.
- Contexto/notas: Coberto — definição agnóstica promovida pra `contexts/common/DDD.md`, exemplo em `go/config/reference/DDD.md` (e `python/config/reference/DDD.md`). README síntese em `go/ddd-em-go/`: o que Go não dá de graça (sem ORM ativo, sem private forçado), anemic domain model como erro comum vindo de Python, quando não vale aplicar DDD tático completo.

### Tópico: Clean Architecture / separação em camadas
- Status: [x]
- Dia do plano: —
- Conteúdo: fronteiras de camada, injeção de dependência sem framework.
- Contexto/notas: Coberto — definição agnóstica promovida pra `contexts/common/CLEAN-ARCHITECTURE.md`, exemplo em `go/config/reference/CLEAN-ARCHITECTURE.md` (e `python/config/reference/CLEAN-ARCHITECTURE.md`). README síntese em `go/clean-architecture-separacao-em-camadas/`: o que faz a regra de dependência "pegar" em Go (interface implícita), teste rápido via grep de import.

### Tópico: Clean Code em Go
- Status: [x]
- Dia do plano: —
- Conteúdo: nomes, funções, comentários (exceção do godoc), formatação, erro, testes, code smells — aplicação Go dos princípios já em `contexts/common/CLEAN-CODE.md`.
- Contexto/notas: Coberto — README síntese em `go/clean-code-em-go/`: ferramenta (gofmt/lint) substituindo convenção manual, erro silencioso mais visível em Go que em Python, interseção com SOLID/DDD (função pequena = SRP quase de graça).

## Módulo 3: Backend em Go

### Tópico: net/http e APIs REST
- Status: [x]
- Dia do plano: 8
- Conteúdo: `net/http`, API REST via stdlib.
- Contexto/notas: Coberto — README em `go/net-http-e-apis-rest/`: `ServeMux` 1.22+ com método+wildcard (dispensa router de terceiro pra caso simples), ordem de `WriteHeader` antes do corpo, link pra design de API em BACKEND-BEST-PRACTICES.md.

### Tópico: Gin (framework web)
- Status: [x]
- Dia do plano: —
- Conteúdo: roteamento, grupos de rota, binding de request, comparação com o que dá pra fazer só com stdlib.
- Contexto/notas: Coberto — README em `go/gin-framework-web/`: quando sair da stdlib, `gin.Context` único, `ShouldBindJSON`, `Group` com middleware compartilhado.

### Tópico: JSON e consumo de APIs externas
- Status: [x]
- Dia do plano: 9
- Conteúdo: JSON, struct tags, consumir APIs externas (base pra integrar com LLMs a partir do Go).
- Contexto/notas: Coberto — README em `go/json-e-consumo-de-apis-externas/`: struct tags/omitempty, Marshal/Unmarshal sem exception, `context.WithTimeout` em chamada externa, `defer resp.Body.Close()`.

### Tópico: Acesso a dados e context.Context
- Status: [x]
- Dia do plano: 10
- Conteúdo: `database/sql`/driver, `context.Context` (cancelamento, timeouts).
- Contexto/notas: Coberto — README em `go/acesso-a-dados-e-context/`: pool embutido (SetMaxOpenConns), Scan campo a campo, ctx propagado em toda cadeia de I/O, cancelamento em cascata.

### Tópico: Migrations de banco de dados
- Status: [x]
- Dia do plano: —
- Conteúdo: ferramentas de migration em Go (ex: golang-migrate/goose), versionamento de schema.
- Contexto/notas: Coberto — README em `go/migrations-de-banco-de-dados/`: golang-migrate/goose/atlas, padrão up/down, schema como fonte da verdade.

### Tópico: Repository pattern na prática
- Status: [x]
- Dia do plano: —
- Conteúdo: interface de repositório no domínio, implementação concreta na infra — amarra direto com Clean Architecture do Módulo 2.
- Contexto/notas: Coberto — README em `go/repository-pattern-na-pratica/`: exemplo completo com implementação Postgres real + fake in-memory pra teste, mesma interface.

### Tópico: Tratamento de erros e exceções em APIs
- Status: [x]
- Dia do plano: —
- Conteúdo: tipos de erro customizados, handler central de erro, mapear erro de domínio para status HTTP.
- Contexto/notas: Coberto — README em `go/tratamento-de-erros-e-excecoes-em-apis/`: `writeError` mapeando sentinel error → status, comparação com exception handler do FastAPI, 3 categorias de erro (validação/negócio/infra).

### Tópico: Middleware e chain de handlers
- Status: [x]
- Dia do plano: —
- Conteúdo: auth, logging, recovery de panic, composição de middlewares.
- Contexto/notas: Coberto — README em `go/middleware-e-chain-de-handlers/`: middleware como função de ordem superior, recover como uso legítimo de panic/recover, chain no Gin (`c.Next()`/`c.Abort()`), ordem importa.

### Tópico: Validação de input
- Status: [x]
- Dia do plano: —
- Conteúdo: struct tags de validação, validar request antes de chegar na regra de negócio.
- Contexto/notas: Coberto — README em `go/validacao-de-input/`: validação manual vs `binding` tag (go-playground/validator via Gin), sempre 400 nunca 500.

### Tópico: Padrões de concorrência para infra
- Status: [x]
- Dia do plano: 11
- Conteúdo: worker pools, rate limiting, pipelines.
- Contexto/notas: Coberto — README em `go/padroes-de-concorrencia-para-infra/`: worker pool com WaitGroup, rate limit com time.Ticker (+ nota sobre x/time/rate), pipeline de estágios encadeados por channel.

### Tópico: Produção — logging, config, error wrapping
- Status: [x]
- Dia do plano: 12
- Conteúdo: logging estruturado, config/env vars, `fmt.Errorf` + `%w`.
- Contexto/notas: Coberto — README em `go/producao-logging-config-error-wrapping/`: `log/slog` (stdlib desde 1.21), correlation ID via `context.WithValue`, config via env falhando cedo no boot, error wrapping revisitado.

### Tópico: Testes (unitários e integração)
- Status: [x]
- Dia do plano: —
- Conteúdo: table-driven tests aplicados a handlers/repositórios, mocks via interface, testes de integração com banco real/testcontainers.
- Contexto/notas: Coberto — README em `go/testes-unitarios-e-integracao/`: `httptest`, fake repo pra unitário vs testcontainers-go pra integração (`testing.Short()`), `go test -race`.

### Tópico: Projeto final — API + LLM
- Status: [x]
- Dia do plano: 13
- Conteúdo: serviço Go que expõe API e chama uma API de LLM (API + infra + IA juntos).
- Contexto/notas: Coberto — README em `go/projeto-final-api-llm/`: esboço de arquitetura completo (domain/llm/infra/http) amarrando todo o Módulo 3. Implementação real ainda não feita, sem exercício gerado.

## Módulo 4: Revisão final

### Tópico: Revisão geral e boas práticas idiomáticas
- Status: [ ]
- Dia do plano: 14
- Conteúdo: gaps, perguntas de entrevista/dia-1, `gofmt`/`golangci-lint`, idiomatic Go.
- Contexto/notas: —
