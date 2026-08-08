# Projeto final — serviço Go/Python com LLM

Síntese de todos os módulos anteriores: um serviço real (Go ou Python) que expõe um agente com RAG e/ou tool calling, numa arquitetura em camadas — não um conceito novo, é a composição de tudo que já foi visto, um módulo puxando o próximo.

## Ponto de partida: o que já existe

O Dia 13 do plano de Go já cobriu a versão mais simples disso — [go/projeto-final-api-llm](../../go/projeto-final-api-llm/): um serviço que expõe `POST /chat` e chama um LLM por trás, com `LLMClient` como interface de domínio e implementação concreta isolada em `infra/`. Este projeto final de IA é a extensão natural dele: o mesmo esqueleto de arquitetura, mas o `LLMClient.Complete` de uma chamada única vira um agente com loop, tools reais e (opcionalmente) busca em RAG.

## Esboço de arquitetura estendido

```
agentservice/
  main.go / main.py
  domain/
    agent.go              // Agent, regra de quando parar o loop — não sabe de HTTP nem de provedor
    tool.go                // interface Tool — contrato que cada tool concreta implementa
  llm/
    client.go               // interface Completer (igual ao Dia 13), agora usado dentro do loop do agente
  tools/
    search_docs.go           // tool que consulta o índice de RAG
    <outras tools>.go
  infra/
    anthropic/
      client.go               // implementa Completer — chamada real ao provedor
    vectorstore/
      pgvector.go              // busca por similaridade — Módulo 3
  http/
    chat_handler.go            // recebe request, roda o agente, responde (streaming opcional)
  observability/
    tracing.go                 // Módulo 8 — log estruturado de cada chamada/tool call
```

## Como cada módulo entra

- **Módulo 1-2 (fundamentos + integração)** — a chamada básica ao LLM, streaming, tool calling — é a base de `llm/client.go` e de como cada `Tool` é descrita via JSON Schema (ver [Tool/function calling e structured output](../tool-function-calling-e-structured-output/)).
- **Módulo 3 (RAG)** — se o agente responde perguntas sobre uma base de conhecimento própria, uma das tools é `search_docs`, que faz embedding da query, busca no vector DB (com chunking e hybrid search já resolvidos na indexação — ver [Chunking, hybrid search e reranking](../chunking-hybrid-search-e-reranking/)), e devolve os trechos relevantes como resultado da tool call. Antes de incluir essa tool, vale revisitar [Quando RAG resolve](../quando-rag-resolve/) — se a base é pequena e estável, um prompt fixo pode bastar e poupa a complexidade inteira do índice.
- **Módulo 4 (agentic loop)** — `domain/agent.go` implementa o loop de [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/): recebe a mensagem, chama o LLM, decide se para ou continua chamando tools, com limite de iterações.
- **Módulo 5 (MCP)** — se alguma capacidade já existe como MCP server (ex: acesso a um sistema interno via server já publicado), o agente conecta como client MCP em vez de reimplementar a tool do zero — ver [Arquitetura MCP](../arquitetura-mcp/).
- **Módulo 6 (harness)** — não se aplica a construir um harness completo aqui (isso é outro nível de produto), mas o modelo de permissão para tools de efeito real (ex: uma tool que escreve dado, não só lê) segue o mesmo raciocínio de [Loop de tool use, contexto e permissions](../loop-de-tool-use-contexto-e-permissions/) — nem toda tool deveria executar sem alguma forma de controle.
- **Módulo 8 (avaliação/segurança/observabilidade)** — antes de considerar o serviço pronto: um dataset mínimo de eval (ver [Evals e LLM-as-judge](../evals-e-llm-as-judge/)) cobrindo os casos principais e pelo menos um caso adversarial de prompt injection (ver [Prompt injection e guardrails](../prompt-injection-e-guardrails/)); tracing de cada chamada com tokens/latência/custo (ver [Observabilidade de LLM em produção](../observabilidade-de-llm-em-producao/)).

## O que muda de verdade em relação ao Dia 13

A arquitetura em camadas é a mesma — `Completer`/`LLMClient` como interface de domínio, implementação isolada em `infra/`, erro tratado com `%w` e HTTP consistente, tudo já coberto em [Integração LLM a partir de Go e Python](../integracao-llm-a-partir-de-go-e-python/). O que é genuinamente novo aqui:
1. Uma chamada única vira um **loop** com estado (histórico crescendo a cada iteração).
2. Tools reais entram no domínio como uma interface própria (`Tool`), não só o `Completer`.
3. Existe uma fronteira explícita de risco — cada tool nova é uma decisão consciente de superfície de ataque (Módulo 8), não só mais uma função.

## Sem exercício pré-gerado

Como os demais tópicos deste vault, a implementação real fica para quando você pedir via `/exercise projeto final servico llm` ou direto no chat — este README é a síntese de referência, não o código rodando.
