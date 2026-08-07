# Plano de Estudo: Go em 2 Semanas

**Contexto:** aprovado em processo seletivo Go/Python + IA. Zero conhecimento em Go, Python nível intermediário. Go será usado em APIs, infra e integração com IA.

**Estratégia:** usar Python como ponte mental — cada conceito novo de Go é explicado comparando com o equivalente (ou a ausência de equivalente) em Python. Foco prático: menos teoria pura, mais código rodando.

---

## Semana 1 — Fundamentos

- [x] **Dia 1** — O que é Go (origem, diferenciais, instalação) + setup (Go toolchain, `go mod`), sintaxe básica: variáveis, tipos, `const`, controle de fluxo (`if`, `for` — Go só tem `for`), funções, múltiplos retornos. Comparar com Python.
- [x] **Dia 2** — Structs e métodos (substituem classes), interfaces (tipagem estrutural implícita — bem diferente de Python/ABC).
- [x] **Dia 3** — Ponteiros (novidade real vs Python), arrays vs slices, maps.
- [x] **Dia 4** — Tratamento de erros idiomático (`error` como valor, sem exceptions), pacotes e organização de módulos.
- [x] **Dia 5** — Concorrência parte 1: goroutines e channels (o maior diferencial de Go, crítico pro papel de infra).
- [x] **Dia 6** — Concorrência parte 2: `sync` (WaitGroup, Mutex), `select`, testes em Go (`testing`, table-driven tests).
- [x] **Dia 7** — Revisão + projeto pequeno: CLI que consolida os conceitos da semana.

## Semana 2 — Aplicado ao trabalho

- [ ] **Dia 8** — `net/http`, construir uma API REST (stdlib primeiro, depois um framework leve tipo chi/gin).
- [ ] **Dia 9** — JSON e struct tags, consumir APIs externas (base pra integrar com APIs de IA/LLM a partir do Go).
- [ ] **Dia 10** — Acesso a dados (`database/sql` ou driver), `context.Context` (cancelamento, timeouts — essencial em Go).
- [ ] **Dia 11** — Padrões de concorrência para infra: worker pools, rate limiting, pipelines.
- [ ] **Dia 12** — Preocupações de produção: logging estruturado, config/env vars, error wrapping (`fmt.Errorf` + `%w`).
- [ ] **Dia 13** — Projeto final: serviço Go que expõe uma API e chama uma API de LLM (junta tudo: API + infra + IA).
- [ ] **Dia 14** — Revisão geral, gaps, perguntas frequentes de dia-1/entrevista técnica, `gofmt`/`golangci-lint`, boas práticas idiomáticas.

---

## Notas de progresso
*(vamos preenchendo aqui conforme avançamos — dúvidas, pontos fracos, exercícios feitos)*

- Semana 1 (Módulo 1 completo) coberta em modo "só explicação" — READMEs gerados em `go/<tópico>/`, sem exercício prático rodado pelo usuário ainda (exceto rascunho de Sintaxe básica em `exercise/`, não confirmado). Ver `contexts/GO-MODULES.md` pra detalhe por tópico.
- Módulo 2 (Boas práticas de engenharia — SOLID/DDD/Clean Architecture em Go) completo, sem dia fixo no cronograma. Também em modo "só explicação", sem exercício.
