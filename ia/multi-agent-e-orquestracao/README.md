# Multi-agent e orquestração

Continuação de [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/) — lá vimos um agente único resolvendo uma tarefa em várias iterações. Aqui, o problema: quando **um** agente com um contexto crescendo indefinidamente para de ser a melhor estrutura, e vale dividir o trabalho entre vários.

## Por que dividir em vários agentes

Um agente único acumula todo o histórico (raciocínio, tool calls, resultados) numa única sequência crescente de contexto. Isso cria dois problemas em tarefas grandes:
- **Poluição de contexto** — detalhe de uma sub-tarefa concluída (ex: o resultado bruto de uma busca extensa) continua ocupando espaço e "distraindo" o modelo em etapas seguintes que não precisam mais dele — o mesmo problema de contexto poluído já visto em [Contexto, sampling e prompt engineering](../contexto-sampling-e-prompt-engineering/).
- **Falta de especialização** — um único system prompt tentando cobrir "pesquise, escreva código, revise, documente" bem para todas essas tarefas ao mesmo tempo tende a ser pior em cada uma do que um prompt focado em uma responsabilidade só.

Dividir em múltiplos agentes especializados, cada um com seu próprio contexto isolado e uma responsabilidade estreita, ataca os dois problemas — é literalmente Single Responsibility Principle ([contexts/common/SOLID.md](../../contexts/common/SOLID.md)) aplicado a agentes: cada um faz uma coisa bem, e a composição do sistema resolve a tarefa maior.

## Padrões de orquestração

- **Supervisor (orchestrator-worker)** — um agente central recebe a tarefa, decide como dividir em sub-tarefas, delega cada uma a um agente/sub-agente especializado, e consolida os resultados. É o padrão mais comum em harnesses de coding agent: um agente principal delega pesquisa de código a um sub-agente de busca, mantendo o contexto principal limpo do ruído da busca. Analogia direta: é o mesmo papel de um `service` que orquestra múltiplos `repository`s numa arquitetura em camadas — coordena, não faz o trabalho fino ele mesmo.

```
Supervisor recebe: "revise este PR e sugira melhorias"
  ├─ delega a Agente A: "busque o histórico de commits relacionado a este arquivo"
  ├─ delega a Agente B: "rode os testes e reporte falhas"
  └─ consolida resultado de A + B → gera revisão final
```

- **Pipeline (sequencial)** — agentes encadeados, saída de um vira entrada do próximo, sem retorno ao anterior. Bom para processos com etapas bem definidas e ordem fixa (ex: extrair dado → validar → formatar → publicar), onde cada etapa é simples o suficiente para não precisar de replanejamento.
- **Paralelo** — múltiplos agentes trabalham simultaneamente em sub-tarefas independentes (ex: pesquisar três tópicos diferentes ao mesmo tempo), e os resultados são agregados só no final. Reduz latência total quando as sub-tarefas não dependem umas das outras — o mesmo raciocínio de paralelizar chamadas independentes já visto em concorrência ([go/concorrencia-goroutines-e-channels](../../go/concorrencia-goroutines-e-channels/)), só que aqui a unidade paralelizada é uma chamada de agente inteira, não uma goroutine.

## Trade-off: complexidade vs ganho

Multi-agent não é "melhor" por padrão — é mais caro (múltiplas chamadas de LLM, possivelmente em paralelo, custam mais no total que uma sequência única) e mais difícil de debugar (falha pode estar na delegação, no agente especializado, ou na consolidação do resultado — mais superfícies de erro que um loop único). A pergunta a fazer antes de dividir:

- **A tarefa tem sub-partes genuinamente independentes ou especializadas?** Se não, um agente único com um bom prompt resolve mais simples e mais barato.
- **O contexto de uma sub-tarefa, se mantido, prejudicaria as outras?** Se sim (ex: uma busca extensa que gera muito ruído), isolar em um sub-agente que devolve só o resumo relevante compensa a complexidade.
- **Existe ganho real de paralelismo?** Se as sub-tarefas são sequenciais por natureza (uma depende do resultado da outra), multi-agent paralelo não ajuda — só adiciona overhead de orquestração sem reduzir latência.

Regra prática, análoga a qualquer decisão de dividir um sistema em serviços menores: comece com um agente único; divida quando o contexto de uma tarefa específica começar a poluir ou degradar as outras, não antes. Abstração/divisão prematura aqui tem o mesmo custo que abstração prematura em qualquer outro código — complexidade paga sem benefício correspondente (ver erro comum já listado em [go/solid-em-go](../../go/solid-em-go/)).
