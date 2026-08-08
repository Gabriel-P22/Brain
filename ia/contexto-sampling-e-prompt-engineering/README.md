# Contexto, sampling e prompt engineering

Pré-requisito: [O que é um LLM](../o-que-e-um-llm/) — este tópico assume que "LLM prevê o próximo token dado o contexto" já está claro, e aprofunda os três controles práticos em cima disso: quanto contexto cabe, como o próximo token é escolhido, e como escrever o texto de entrada para guiar essa escolha.

## Context window

O **context window** é o número máximo de tokens (entrada + saída somados) que o modelo processa em uma chamada. Não é uma configuração arbitrária — é uma limitação estrutural: o mecanismo de attention (visto no tópico anterior) compara cada token com todos os outros, então custo computacional cresce com o tamanho do contexto. Modelos atuais variam de ~128k a mais de 1M tokens, mas "cabe na janela" não é o mesmo que "o modelo usa bem" — em janelas muito grandes, informação no meio do contexto tende a receber menos atenção efetiva do que informação no início ou no fim (efeito às vezes chamado de "lost in the middle").

Analogia com Python: é parecido com passar um payload gigante para uma função — tecnicamente aceita, mas se a lógica que consome esse payload não foi pensada para volume grande, processar 200 campos quando só 5 importam degrada qualidade da resposta, não só custo. Contexto grande não substitui contexto **relevante**.

Consequência prática de arquitetura: **contexto é um recurso finito e caro**, análogo a memória em um sistema com orçamento — daí a existência de RAG (buscar só o trecho relevante em vez de mandar tudo, ver [Quando RAG resolve](../quando-rag-resolve/)) e de prompt caching (ver [APIs de LLM e streaming](../apis-de-llm-e-streaming/)).

## Sampling: como o próximo token é escolhido

O modelo não devolve "o token certo" — devolve uma distribuição de probabilidade sobre todo o vocabulário para o próximo token. Sampling é a estratégia de escolher um token real a partir dessa distribuição.

- **`temperature`** — controla o quão "achatada" ou "afiada" fica a distribuição antes de amostrar. `temperature=0` tende ao token de maior probabilidade sempre (saída mais determinística e repetível — mas não 100% garantida dependendo do provedor); `temperature` alta (ex: 1.0+) dá mais chance a tokens menos prováveis, produzindo saída mais "criativa"/variada e também mais sujeita a erro.
- **`top_p` (nucleus sampling)** — em vez de considerar o vocabulário inteiro, restringe a amostragem ao menor conjunto de tokens cuja probabilidade acumulada passa de `p` (ex: `top_p=0.9` considera só os tokens mais prováveis até somar 90%). Costuma ser combinado com `temperature`, não usado sozinho.

Regra prática: **tarefa determinística (extração de dado, código, classificação) → temperature baixa**; **tarefa geradora/criativa (brainstorm, variações de texto) → temperature mais alta**. Não existe "melhor valor" universal — é o mesmo tipo de trade-off de configuração de sistema que já é familiar (ex: nível de log, tamanho de pool de conexão): depende do que a tarefa exige.

## Prompt engineering: técnicas

- **System prompt vs user prompt** — o `system prompt` define papel, restrições e comportamento persistente do modelo para toda a conversa (equivalente a configuração/injeção de dependência feita uma vez); o `user prompt` é a entrada específica daquela chamada. Separar os dois evita repetir instrução de comportamento em toda mensagem e reduz risco de o usuário final sobrescrever regra de negócio via prompt injection (ver [Prompt injection e guardrails](../prompt-injection-e-guardrails/)).
- **Zero-shot** — pedir a tarefa direto, sem exemplo. Funciona para tarefas comuns/bem representadas no treino.
- **Few-shot** — incluir 2-5 exemplos de entrada/saída no prompt antes da tarefa real. É o jeito mais barato de "ensinar" um formato específico sem fine-tuning — análogo a passar casos de teste como especificação em vez de descrever a regra em prosa.
- **Chain-of-thought** — pedir para o modelo gerar passos de raciocínio antes da resposta final ("pense passo a passo"). Como visto no tópico anterior, isso não é o modelo "decidindo pensar" — é forçar geração de tokens intermediários que mudam a distribuição de probabilidade da resposta final, geralmente para melhor em tarefas que envolvem múltiplas etapas lógicas.

```
System: Você é um extrator de dados. Responda só em JSON válido, sem texto extra.
User: "João Silva, 32 anos, engenheiro" → extraia nome, idade, profissão.
```

Isso já é o embrião do que vira **structured output** de verdade (schema validado, não só "peça JSON e torça") — aprofundado em [Tool/function calling e structured output](../tool-function-calling-e-structured-output/).

## Prompt engineering vs context engineering

"Prompt engineering" é escrever bem *a instrução* de uma chamada isolada. "Context engineering" é um escopo maior: decidir **o que** entra no contexto em primeiro lugar — que documentos recuperar (RAG), que histórico de conversa manter ou resumir, que resultado de tool call incluir, o que descartar. Em um sistema real (agente, chat com histórico longo, RAG), a maior parte do trabalho de engenharia não é redigir a instrução — é gerenciar o orçamento de tokens do contexto de forma que o modelo veja só o que é relevante para aquela chamada. É o mesmo tipo de disciplina de "responsabilidade única" aplicada ao que entra numa função: um contexto poluído com informação irrelevante degrada a resposta do mesmo jeito que uma função com parâmetros demais degrada legibilidade — ver [contexts/common/CLEAN-CODE.md](../../contexts/common/CLEAN-CODE.md).
