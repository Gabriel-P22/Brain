# Loop de tool use, contexto e permissions

Continuação de [O que é um harness](../o-que-e-um-harness/) — aqui, os três mecanismos internos que separam um agente de demonstração ([Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)) de um harness usável em produção no dia a dia.

## O loop de tool use, com mais peça

O loop básico (modelo pede tool → código executa → resultado volta) continua sendo o mesmo de [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/), mas um harness real adiciona camadas em volta de cada chamada de tool:

```
modelo pede tool call
        │
        ▼
verificação de permissão (auto-allow? pede confirmação? bloqueado?)
        │
        ▼
execução (com timeout, sandboxing, captura de erro)
        │
        ▼
resultado formatado volta pro histórico
        │
        ▼
verificação: contexto está perto do limite? → compressão se necessário
        │
        ▼
próxima chamada ao modelo
```

Cada uma dessas camadas existe por um motivo de produto/segurança específico, detalhado abaixo.

## Compressão/sumarização de contexto

Uma sessão longa (como esta mesma conversa) acumula histórico além do que cabe confortavelmente na context window (ver [Contexto, sampling e prompt engineering](../contexto-sampling-e-prompt-engineering/)). Um harness monitora o tamanho do contexto acumulado e, ao se aproximar do limite, substitui parte do histórico antigo por um **resumo** gerado (geralmente por uma chamada ao próprio LLM pedindo para condensar o que já aconteceu), preservando decisões e fatos relevantes e descartando detalhe operacional que não importa mais (ex: o conteúdo bruto retornado por uma tool call de 5 mensagens atrás).

Isso é o mesmo trade-off já visto em [Chunking, hybrid search e reranking](../chunking-hybrid-search-e-reranking/) entre granularidade e cobertura: resumir demais perde detalhe que pode ser necessário depois; resumir de menos não libera espaço suficiente. Harnesses de produção calibram isso empiricamente, e normalmente preservam certas âncoras à parte da compressão (ex: arquivos de instrução como `CLAUDE.md`, que continuam carregados independente de quanto o histórico de conversa é comprimido).

## Modelo de permissão

Como o modelo pode pedir para executar qualquer ação (editar arquivo, rodar comando de shell, fazer request de rede) e é, por definição, uma fonte de decisão não 100% confiável (ver [Prompt injection e guardrails](../prompt-injection-e-guardrails/)), um harness precisa de uma política clara de quando uma ação executa sozinha e quando para para confirmação humana:

- **Auto-allow** — ações de baixo risco e reversíveis (ler um arquivo, rodar um comando de leitura como `git status`) executam sem interromper o fluxo — pedir confirmação para tudo tornaria o harness inutilizável na prática.
- **Prompt de confirmação** — ações com efeito colateral relevante ou difícil de reverter (escrever em arquivo, fazer commit, deletar algo, chamar uma API externa que muda estado) param e pedem aprovação explícita do usuário antes de executar.
- **Sandboxing** — mesmo ações permitidas rodam com escopo restrito (ex: comando de shell limitado ao diretório do projeto, sem acesso irrestrito ao resto do sistema) — princípio de menor privilégio, mesmo raciocínio já visto para tool design em [Tool/function calling e structured output](../tool-function-calling-e-structured-output/), aplicado agora ao ambiente de execução inteiro, não só ao schema de uma tool.

Esse modelo de permissão é, na prática, análogo a controle de acesso em qualquer sistema — a diferença é que aqui quem está "pedindo" a ação é o output de um modelo probabilístico, não uma requisição de um usuário autenticado com intenção clara, então o nível de desconfiança por padrão é maior.

## Por que isso é o núcleo de "harness bem feito"

A diferença prática entre um agente de demonstração (o loop simples de [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)) e um harness usado em produção por milhares de devs todo dia não é o modelo por trás — é o cuidado de engenharia nessas três camadas: gerenciar contexto sem perder informação relevante, dar controle de permissão granular sem tornar o uso tedioso, e manter tudo isso confiável mesmo quando o modelo erra ou o ambiente falha. É engenharia de produto/sistema aplicada em cima de um componente probabilístico — a mesma disciplina de tratamento de erro, timeout e princípio de menor privilégio já familiar de qualquer sistema em produção (ver [contexts/common/BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md)), só que a "dependência externa não confiável" agora é o próprio modelo de linguagem.
