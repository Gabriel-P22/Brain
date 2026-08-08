# O que é um harness

## Modelo vs produto construído em volta dele

Um LLM sozinho (a API, ver [APIs de LLM e streaming](../apis-de-llm-e-streaming/)) só faz uma coisa: recebe texto, devolve texto (ou uma tool call, ver [Tool/function calling e structured output](../tool-function-calling-e-structured-output/)). Ele não lê arquivo, não edita código, não lembra da conversa de ontem, não pede permissão antes de rodar um comando. Tudo isso — acesso a ferramentas reais, memória entre sessões, controle de permissão, gerenciamento de contexto — é construído **por fora** do modelo, na camada de aplicação. Essa camada é o **harness**.

Claude Code (o que está executando esta sessão), Cursor, Windsurf, GitHub Copilot Workspace são todos harnesses: produtos que dão a um LLM acesso a um conjunto de tools (ler/escrever arquivo, rodar comando de shell, buscar na web), implementam o loop agente (ver [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)), e resolvem os problemas de engenharia de produto que tornam isso usável e seguro no dia a dia: permissão, sandboxing, compressão de contexto, integração com o resto do fluxo de trabalho do dev.

## Analogia com o que você já conhece

A relação entre LLM e harness é parecida com a relação entre um motor e o carro construído ao redor dele — o motor sozinho não anda, não tem direção, não tem freio. Em termos de arquitetura já familiar: o LLM é a "infra" mais fundamental (um serviço externo de geração de texto), e o harness é a aplicação inteira em camadas construída em cima — orquestra chamadas ao modelo, expõe tools como fronteira controlada de efeito colateral (mesmo raciocínio de [Tool/function calling e structured output](../tool-function-calling-e-structured-output/)), e decide o que entra em cada chamada.

## O que um harness precisa resolver, além do loop básico

- **Ferramentas reais** — dar ao modelo acesso a ler/escrever arquivo, rodar shell, buscar na web, chamar MCP servers ([Arquitetura MCP](../arquitetura-mcp/)) — cada uma implementada com cuidado de segurança (validação, sandboxing) porque o modelo é, do ponto de vista de confiança, uma fonte externa não confiável decidindo o que executar (ver [Prompt injection e guardrails](../prompt-injection-e-guardrails/)).
- **Memória/persistência** — uma sessão de chat termina, mas o trabalho continua depois. Harnesses como Claude Code resolvem isso com arquivos de projeto (`CLAUDE.md`), histórico de sessão, ou sistemas de memória explícitos — porque o modelo, por si, não retém nada entre chamadas de API (ver [O que é um LLM](../o-que-e-um-llm/)).
- **Modelo de permissão** — decidir quando uma ação do modelo (editar arquivo, rodar comando) pode executar automaticamente e quando precisa de confirmação humana antes. Aprofundado no próximo tópico.
- **Gerenciamento de contexto** — uma sessão longa acumula histórico além do que cabe (ou faz sentido) manter na janela de contexto; o harness decide o que resumir, compactar ou descartar. Também aprofundado no próximo tópico.

## Onde isso conecta com o resto do módulo

Este tópico é a categoria; [Loop de tool use, contexto e permissions](../loop-de-tool-use-contexto-e-permissions/) é o mecanismo interno de como um harness resolve isso na prática. E, olhando para trás no plano: um harness de coding agent é, estruturalmente, o mesmo agente de [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/) — só que com mais tools, mais cuidado de produto, e rodando continuamente numa sessão longa em vez de resolver uma tarefa isolada de uma vez.
