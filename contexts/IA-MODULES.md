# IA-MODULES

Conteúdo por tópico de IA — acumula contexto (notas, dúvidas, erros recorrentes) conforme o tópico é estudado, independente de quando isso acontece. Sem cronograma fixo (diferente do Go): cada tópico é estudado no ritmo que fizer sentido, via `/topic`.

Padrão do vault: cada assunto grande ganha seu próprio `*-MODULES.md` (este é o de IA; segue o mesmo formato do `GO-MODULES.md`).

Ordem pensada como panorama de mercado, do conceito de base até prática de engenharia: fundamentos de LLM → integração prática → RAG → agentes → MCP → harness/coding agents → spec-driven development → avaliação/segurança/observabilidade → projeto aplicado (amarra tudo com Go/Python, ver `contexts/GO_STUDY_PLAN.md` Dia 9/13).

Diferente de Go/Python, este assunto não tem `config/reference/{SOLID,CLEAN-CODE,DDD,CLEAN-ARCHITECTURE}.md` — não é uma linguagem de programação com idioma próprio pra esses princípios. Quando um tópico tocar em design de software (ex: como estruturar um serviço que integra LLM), a referência é cruzada pra `contexts/common/` e pro exemplo Go/Python já existente, não duplicada aqui.

---

## Módulo 1: Fundamentos de LLM

### Tópico: O que é um LLM
- Status: [x]
- Dia do plano: —
- Conteúdo: o que é um modelo de linguagem, tokens, embeddings, transformers/attention em visão de alto nível (sem matemática pesada) — o suficiente pra entender por que LLM se comporta do jeito que se comporta (previsão de próximo token, não "raciocínio" no sentido clássico).
- Contexto/notas: README criado em `ia/o-que-e-um-llm/`. Cobre previsão de próximo token, tokens/tokenização, embeddings em alto nível, transformers/attention sem matemática pesada, e por que isso explica alucinação e chain-of-thought.

### Tópico: Contexto, sampling e prompt engineering
- Status: [x]
- Dia do plano: —
- Conteúdo: context window (o que é, por que tem limite, custo de token), temperature/top-p/sampling, técnicas de prompt (few-shot, chain-of-thought, system prompt vs user prompt), diferença entre "prompt engineering" e "context engineering".
- Contexto/notas: README criado em `ia/contexto-sampling-e-prompt-engineering/`. Cobre context window (e efeito "lost in the middle"), temperature/top_p, zero/few-shot, chain-of-thought, system vs user prompt, e a distinção prompt engineering vs context engineering.

## Módulo 2: Integração prática com LLM

### Tópico: APIs de LLM e streaming
- Status: [x]
- Dia do plano: —
- Conteúdo: uso de SDK (Anthropic/OpenAI), request/response básico, streaming de resposta, prompt caching, custo/pricing por token.
- Contexto/notas: README criado em `ia/apis-de-llm-e-streaming/`. Cobre request/response básico (SDK Anthropic), streaming via SSE, prompt caching e pricing por token (entrada/saída/cache).

### Tópico: Tool/function calling e structured output
- Status: [x]
- Dia do plano: —
- Conteúdo: como um LLM "chama função" (na prática, é o modelo devolvendo JSON estruturado pra você executar — não é o modelo rodando código), JSON schema de tool, structured output/response format, por que isso é a base de todo agente.
- Contexto/notas: README criado em `ia/tool-function-calling-e-structured-output/`. Deixa claro que o modelo só devolve JSON, quem executa é o código; cobre JSON Schema como contrato, structured output via tool forçada, e tool como fronteira de infra (liga com Clean Architecture).

### Tópico: Integração LLM a partir de Go e Python
- Status: [x]
- Dia do plano: —
- Conteúdo: chamando API de LLM a partir de um serviço Go (`net/http`, `context.Context` pra timeout/cancelamento) e de Python, tratamento de erro/retry/rate limit, onde isso se encaixa numa arquitetura em camadas (LLM como detalhe de infra, não vazando pro domínio — liga com Clean Architecture já visto em Go).
- Contexto/notas: README criado em `ia/integracao-llm-a-partir-de-go-e-python/`. Completer/LLMClient como interface de domínio (DIP), exemplo completo em Go (net/http + context.Context) e Python (SDK + retry/backoff), erro tratado com %w / exceções.

## Módulo 3: RAG (Retrieval-Augmented Generation)

### Tópico: Embeddings e vector databases
- Status: [x]
- Dia do plano: —
- Conteúdo: embedding como representação vetorial de significado, similaridade (cosseno), o que é um vector DB (pgvector, Pinecone, Qdrant, etc.) e como difere de busca por índice tradicional.
- Contexto/notas: README criado em `ia/embeddings-e-vector-databases/`. Embedding como vetor de significado, similaridade de cosseno, pgvector vs Pinecone/Qdrant/Weaviate, diferença entre índice vetorial e índice relacional tradicional.

### Tópico: Chunking, hybrid search e reranking
- Status: [x]
- Dia do plano: —
- Conteúdo: estratégias de chunking de documento, busca híbrida (vetorial + keyword/BM25), reranking do resultado antes de mandar pro LLM.
- Contexto/notas: README criado em `ia/chunking-hybrid-search-e-reranking/`. Estratégias de chunking (fixed-size, semantic, recursive), hybrid search (vetorial + BM25), reranking como segunda passada, pipeline completo de retrieval.

### Tópico: Quando RAG resolve (e quando não resolve)
- Status: [x]
- Dia do plano: —
- Conteúdo: RAG vs fine-tuning vs prompt longo/context caching — trade-off de custo, latência, atualização de dado, e os casos onde RAG é solução errada pro problema.
- Contexto/notas: README criado em `ia/quando-rag-resolve/`. RAG vs fine-tuning vs prompt longo/caching em tabela de trade-off, e os casos onde RAG é a solução errada (base pequena/estável, pergunta é sobre dado estruturado).

## Módulo 4: Agentes e tool use

### Tópico: Agentic loop (ReAct) e planning
- Status: [x]
- Dia do plano: —
- Conteúdo: o loop básico de um agente (pensar → chamar ferramenta → observar resultado → repetir), padrão ReAct, planejamento explícito vs implícito.
- Contexto/notas: README criado em `ia/agentic-loop-react-e-planning/`. Loop pensar→agir→observar→repetir com código completo, padrão ReAct, planning explícito vs implícito, limites do loop (custo, erro composto, max_iters).

### Tópico: Multi-agent e orquestração
- Status: [x]
- Dia do plano: —
- Conteúdo: quando dividir trabalho entre múltiplos agentes especializados em vez de um agente só, padrões de orquestração (supervisor, pipeline, paralelo), trade-off de complexidade vs ganho.
- Contexto/notas: README criado em `ia/multi-agent-e-orquestracao/`. Por que dividir (poluição de contexto, especialização — SRP aplicado a agentes), padrões supervisor/pipeline/paralelo, trade-off de complexidade vs ganho.

## Módulo 5: MCP (Model Context Protocol)

### Tópico: Arquitetura MCP (client/server/host)
- Status: [x]
- Dia do plano: —
- Conteúdo: o problema que o MCP resolve (integração N-pra-N entre modelo e ferramenta vira N-pra-1 via protocolo padronizado), papel de client/server/host, comparação com o que existia antes (tool calling ad-hoc por app).
- Contexto/notas: README criado em `ia/arquitetura-mcp/`. Problema M×N → M+N que o MCP resolve, papéis host/client/server, comparação com tool calling ad-hoc, nota de que o Claude Code é host MCP.

### Tópico: Tools, resources e prompts em MCP
- Status: [x]
- Dia do plano: —
- Conteúdo: os três primitivos de um MCP server (tools, resources, prompts), como escrever um server simples, como um harness (ex: Claude Code) consome isso.
- Contexto/notas: README criado em `ia/tools-resources-e-prompts-mcp/`. Os três primitivos explicados e diferenciados, exemplo de server mínimo com FastMCP, como um harness consome isso, transporte stdio vs HTTP.

## Módulo 6: Harness / coding agents

### Tópico: O que é um harness
- Status: [x]
- Dia do plano: —
- Conteúdo: definição de "harness" (a camada que dá ao LLM acesso a ferramentas, memória, permissões — Claude Code, Cursor, Windsurf, etc.), diferença entre o modelo em si e o produto construído em volta dele.
- Contexto/notas: README criado em `ia/o-que-e-um-harness/`. Define harness como a camada de produto construída em volta do LLM (tools, memória, permissão, contexto), com Claude Code como exemplo concreto.

### Tópico: Loop de tool use, contexto e permissions
- Status: [x]
- Dia do plano: —
- Conteúdo: como um harness gerencia o loop de chamadas de ferramenta, compressão/sumarização de contexto quando a conversa cresce, modelo de permissão (auto-allow, prompt de confirmação, sandboxing).
- Contexto/notas: README criado em `ia/loop-de-tool-use-contexto-e-permissions/`. Loop de tool use com camadas de permissão/sandboxing, compressão de contexto em sessão longa, modelo de permissão (auto-allow/confirmação/sandbox).

## Módulo 7: Spec-driven development

### Tópico: Spec-driven vs vibe coding
- Status: [x]
- Dia do plano: —
- Conteúdo: o que é desenvolvimento orientado a especificação (escrever a spec antes do código, deixar o agente implementar a partir dela) vs "vibe coding" (iterar direto em cima do código sem spec formal), trade-off de velocidade vs controle/rastreabilidade.
- Contexto/notas: README criado em `ia/spec-driven-vs-vibe-coding/`. Define os dois estilos, trade-off velocidade vs rastreabilidade, onde traçar a linha na prática, conexão com plan mode de agentes.

### Tópico: Ferramentas de mercado
- Status: [x]
- Dia do plano: —
- Conteúdo: panorama de ferramentas (spec-kit e afins), como isso se relaciona com plan mode de agentes de código, onde isso converge com prática de engenharia que o usuário já conhece (requirements doc, RFC, design doc).
- Contexto/notas: README criado em `ia/ferramentas-de-mercado/`. Espectro de formalização (plan mode embutido → templates/skills → spec-kit e afins), convergência com RFC/design doc, exemplo do próprio vault (.claude/commands/, contexts/plans/).

## Módulo 8: Avaliação, segurança e observabilidade

### Tópico: Evals e LLM-as-judge
- Status: [x]
- Dia do plano: —
- Conteúdo: como avaliar qualidade de output de LLM de forma sistemática (dataset de eval, métricas, LLM como avaliador de outro LLM), por que isso substitui "parece bom" por medição.
- Contexto/notas: README criado em `ia/evals-e-llm-as-judge/`. Anatomia de um eval (dataset/métrica/score), LLM-as-judge com critério estreito e explícito, por que isso substitui "parece bom" por medição reprodutível.

### Tópico: Prompt injection e guardrails
- Status: [x]
- Dia do plano: —
- Conteúdo: prompt injection (direta e indireta via dado externo/RAG), por que é um problema de segurança real e não só teórico, guardrails (validação de input/output, sandboxing de tool, princípio de menor privilégio aplicado a agente).
- Contexto/notas: README criado em `ia/prompt-injection-e-guardrails/`. Injection direta vs indireta (via RAG/tool result), por que é estrutural e não um bug corrigível, guardrails em camadas (validação, sandboxing, menor privilégio, confirmação humana).

### Tópico: Observabilidade de LLM em produção
- Status: [x]
- Dia do plano: —
- Conteúdo: tracing de chamada de LLM/agente, logging de prompt/resposta/custo, o que monitorar diferente de uma API tradicional (tokens, latência por chamada, taxa de erro de tool call).
- Contexto/notas: README criado em `ia/observabilidade-de-llm-em-producao/`. Métricas específicas de LLM (tokens, latência por chamada, taxa de erro de tool call, stop_reason), tracing de agente com trace_id, logging de prompt/resposta/custo.

## Módulo 9: Projeto aplicado

### Tópico: Projeto final — serviço Go/Python com LLM
- Status: [x]
- Dia do plano: —
- Conteúdo: amarrar os módulos anteriores num serviço real (API em Go ou Python que expõe um agente com RAG e/ou tool calling), aplicando a arquitetura em camadas já vista no Módulo 2/3 de Go.
- Contexto/notas: README criado em `ia/projeto-final-servico-go-python-llm/`. Síntese de todos os módulos, arquitetura estendida a partir do go/projeto-final-api-llm (Dia 13) — Completer + loop de agente + tools + RAG opcional + observabilidade.

---

## Notas de progresso

*(vamos preenchendo aqui conforme avançamos — dúvidas, pontos fracos, exercícios feitos)*

- Estrutura criada em 2026-08-08, ainda sem nenhum tópico estudado. 9 módulos, 20 tópicos, sem cronograma fixo (diferente do plano de Go de 14 dias) — ritmo livre via `/topic`.
- **2026-08-08 — todos os 9 módulos (20 tópicos) gerados de uma vez**, modo "só explicação" (sem exercício rodado, como no padrão do resto do vault — exercício é opt-in via `/exercise`). READMEs em `ia/<pasta-do-tópico>/`, todos cruzando com Go/Python já estudados e com `contexts/common/{SOLID,CLEAN-CODE,DDD,CLEAN-ARCHITECTURE,BACKEND-BEST-PRACTICES}.md` sempre que o tópico toca em design de software.
- **Pendência real, igual ao que já vale para Go:** nada disso foi praticado ainda — é leitura/teoria. Quando fizer sentido, valeria rodar `/exercise` nos tópicos mais aplicáveis à entrevista (tool calling, RAG básico, integração Go/Python com LLM) e, futuramente, montar o projeto final de verdade (`ia/projeto-final-servico-go-python-llm/`) estendendo o `go/projeto-final-api-llm` do Dia 13.
