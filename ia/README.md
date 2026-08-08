# IA

Panorama de IA aplicada — LLMs, RAG, agentes, MCP, harness de coding agents, spec-driven development, avaliação/segurança — na direção do cargo Go/Python + IA. Sem cronograma fixo (diferente do plano de Go): conteúdo em [contexts/IA-MODULES.md](../contexts/IA-MODULES.md), este README só lista a ordem de estudo e aponta pra pasta de cada tópico.

Cada pasta de tópico tem seu próprio `README.md` (explicação, gerado por `/topic`) e uma subpasta `exercise/` (código de prática opcional, gerado por `/exercise`, fora do git). Diferente de `go/` e `python/`, não existe `config/reference/` aqui — IA não é uma linguagem de programação com idioma próprio de SOLID/DDD/Clean Code; quando um tópico tocar em design de software, a referência cruza pra `contexts/common/` e pro exemplo já existente em Go/Python.

## Módulo 1: Fundamentos de LLM

0. [O que é um LLM](o-que-e-um-llm/)
1. [Contexto, sampling e prompt engineering](contexto-sampling-e-prompt-engineering/)

## Módulo 2: Integração prática com LLM

2. [APIs de LLM e streaming](apis-de-llm-e-streaming/)
3. [Tool/function calling e structured output](tool-function-calling-e-structured-output/)
4. [Integração LLM a partir de Go e Python](integracao-llm-a-partir-de-go-e-python/)

## Módulo 3: RAG (Retrieval-Augmented Generation)

5. [Embeddings e vector databases](embeddings-e-vector-databases/)
6. [Chunking, hybrid search e reranking](chunking-hybrid-search-e-reranking/)
7. [Quando RAG resolve (e quando não resolve)](quando-rag-resolve/)

## Módulo 4: Agentes e tool use

8. [Agentic loop (ReAct) e planning](agentic-loop-react-e-planning/)
9. [Multi-agent e orquestração](multi-agent-e-orquestracao/)

## Módulo 5: MCP (Model Context Protocol)

10. [Arquitetura MCP (client/server/host)](arquitetura-mcp/)
11. [Tools, resources e prompts em MCP](tools-resources-e-prompts-mcp/)

## Módulo 6: Harness / coding agents

12. [O que é um harness](o-que-e-um-harness/)
13. [Loop de tool use, contexto e permissions](loop-de-tool-use-contexto-e-permissions/)

## Módulo 7: Spec-driven development

14. [Spec-driven vs vibe coding](spec-driven-vs-vibe-coding/)
15. [Ferramentas de mercado](ferramentas-de-mercado/)

## Módulo 8: Avaliação, segurança e observabilidade

16. [Evals e LLM-as-judge](evals-e-llm-as-judge/)
17. [Prompt injection e guardrails](prompt-injection-e-guardrails/)
18. [Observabilidade de LLM em produção](observabilidade-de-llm-em-producao/)

## Módulo 9: Projeto aplicado

19. [Projeto final — serviço Go/Python com LLM](projeto-final-servico-go-python-llm/)
