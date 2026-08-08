# Quando RAG resolve (e quando não resolve)

Fecha o Módulo 3: [Embeddings e vector databases](../embeddings-e-vector-databases/) e [Chunking, hybrid search e reranking](../chunking-hybrid-search-e-reranking/) mostraram **como** construir RAG. Este tópico é sobre **quando vale a pena** — a pergunta que precisa vir antes de montar o pipeline, não depois.

## O que RAG resolve, de fato

RAG (Retrieval-Augmented Generation) existe para um problema específico: o LLM não tem acesso, no seu treino, a dado **privado** (documentação interna, base de clientes) nem a dado que muda **depois** do corte de treino (preço atualizado, política nova, estoque). RAG resolve isso buscando o trecho relevante em tempo de consulta e injetando no prompt — o modelo responde com base em texto que você forneceu, não no que "lembra" de ter visto no treino. É a aplicação direta do que foi visto em [O que é um LLM](../o-que-e-um-llm/): reduzir a chance de alucinação fornecendo grounding externo em vez de confiar na memória estatística do modelo.

## RAG vs fine-tuning vs prompt longo/context caching

Três formas diferentes de dar "conhecimento extra" a um LLM, com trade-offs bem diferentes:

| Abordagem | Quando faz sentido | Custo/complexidade |
|---|---|---|
| **RAG** | Base de conhecimento grande, muda com frequência, precisa de citação/rastreabilidade da fonte | Pipeline de indexação + busca (Módulo 3 inteiro), mas dado atualiza sem retreinar nada |
| **Fine-tuning** | Ensinar um **estilo**/formato/comportamento consistente, não fatos que mudam — ex: sempre responder num tom específico, sempre seguir um formato de saída | Caro e lento para atualizar (precisa retreinar a cada mudança de dado) — péssima opção para dado que muda |
| **Prompt longo + caching** | Base de conhecimento pequena o suficiente para caber inteira no context window (dezenas de páginas, não milhares) | Mais simples que montar RAG — sem índice, sem busca — mas não escala em volume, e sem prompt caching (ver [APIs de LLM e streaming](../apis-de-llm-e-streaming/)) fica caro reprocessar tudo a cada chamada |

Erro comum: escolher fine-tuning para "ensinar fatos da empresa" — fine-tuning ajusta como o modelo *se comporta*, não funciona bem como substituto de banco de dados de fatos, e fica desatualizado no minuto em que o dado muda. Se o objetivo é "responder com base em documento X", isso é RAG (ou prompt longo, se X for pequeno) — não fine-tuning.

## Trade-offs reais: custo, latência, atualização

- **Latência**: RAG adiciona uma etapa inteira (busca + reranking) antes até de chamar o LLM — para uma aplicação onde resposta instantânea importa mais que precisão factual profunda, isso pode não compensar.
- **Custo de engenharia**: manter um pipeline de RAG em produção é manter mais um sistema com estado (índice precisa ser atualizado quando o documento fonte muda, re-chunkar quando a estratégia de chunking evolui, monitorar qualidade de retrieval) — não é "configura uma vez e esquece".
- **Atualização de dado**: aqui RAG ganha disparado de fine-tuning — atualizar a base é só reindexar o documento novo, sem retreinar nada.
- **Qualidade depende de todo o pipeline anterior**: se o chunking corta mal ou a busca não recupera o trecho certo, o LLM recebe contexto errado e responde errado com a mesma confiança de sempre — RAG não elimina alucinação, só reduz a chance dela ao fornecer grounding; um retrieval ruim ainda produz resposta ruim (e convincente).

## Quando RAG é a solução errada

- **Base de conhecimento pequena e estável** — cabe inteira no prompt com folga, não muda com frequência: monte RAG e você criou complexidade (índice, busca, reranking) para resolver um problema que um prompt longo + caching já resolvia mais simples.
- **A pergunta não é sobre um documento, é sobre computar algo** — "qual o total de vendas do último trimestre filtrado por região" não é uma pergunta de busca semântica, é uma query estruturada; a resposta correta é uma tool que consulta o banco (ver [Tool/function calling e structured output](../tool-function-calling-e-structured-output/)), não um chunk de texto recuperado por similaridade.
- **Precisão exata é obrigatória e o corpus é pequeno** — para um handful de regras de negócio que cabem numa tabela, um lookup direto (ou até uma regra hardcoded) é mais confiável e mais barato que depender de retrieval aproximado.

A pergunta certa antes de montar qualquer pipeline de RAG: "esse problema é sobre encontrar o trecho relevante em muito texto não estruturado, que muda com frequência?" — se sim, RAG. Se a resposta for "não, é sobre dado estruturado" ou "não, cabe tudo num prompt", a solução mais simples resolve, e complexidade adicional sem necessidade é exatamente o tipo de over-engineering que já é evitado em qualquer outra decisão de arquitetura (ver [contexts/common/CLEAN-ARCHITECTURE.md](../../contexts/common/CLEAN-ARCHITECTURE.md)).
