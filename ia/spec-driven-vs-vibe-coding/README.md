# Spec-driven vs vibe coding

## Duas formas de usar um coding agent

Com um harness capaz de editar código de verdade (Módulo 6), surgem dois estilos bem diferentes de condução:

- **Vibe coding** — iterar direto em cima do código, pedido por pedido, sem um documento de especificação formal antes. "Adiciona um botão aqui", "agora corrige esse bug", "muda o texto disso" — cada instrução é curta, o agente interpreta e executa, você revisa o resultado e ajusta na próxima instrução. Rápido para prototipagem e mudanças pequenas, mas a intenção completa nunca fica registrada em lugar nenhum — só existe implícita na sequência de prompts.
- **Spec-driven development** — escrever a especificação (o quê construir, requisitos, critérios de aceite, casos de borda) **antes** de qualquer código, como um documento explícito, e então deixar o agente implementar a partir dessa spec. A spec vira o artefato de referência: se a implementação diverge do esperado, você compara contra a spec, não contra sua memória do que pediu three prompts atrás.

## Trade-off: velocidade vs controle/rastreabilidade

Vibe coding é mais rápido para tarefas pequenas e bem contidas, onde o custo de escrever uma spec formal excede o valor dela — corrigir um bug óbvio, ajustar um estilo visual, uma mudança de uma linha. Spec-driven paga o custo inicial de escrever a especificação, mas ganha de volta em tarefas maiores: **rastreabilidade** (dá para auditar depois se a implementação cobre todo requisito), **paralelismo** (um agente — ou pessoa — consegue implementar a partir da spec sem precisar da sequência completa de decisões que levou até ali), e **revisão antes de código escrito** (um erro de entendimento pego na spec custa uma edição de texto; o mesmo erro pego só depois do código pronto custa reescrever código).

Isso não é uma ideia nova trazida por LLM — é o mesmo trade-off entre "codar direto" e "escrever requirements/RFC antes" que já existe em engenharia de software tradicional, só que agora o "implementador" a partir da spec pode ser um agente, não só um humano. Quem já trabalhou com um RFC ou design doc antes de abrir um PR grande já viveu esse trade-off — LLM só reduz o custo de tempo de implementação depois que a spec está pronta, tornando o "vale a pena escrever spec" mais frequentemente verdadeiro do que era antes (porque a implementação em si ficou mais barata em tempo, o ROI de investir em uma spec boa subiu).

## Onde a linha se traça, na prática

- Mudança pequena, escopo óbvio, baixo custo de erro → vibe coding direto.
- Feature nova, múltiplos arquivos/componentes afetados, ambiguidade real sobre comportamento esperado, ou trabalho que outra pessoa (ou agente) vai continuar depois → vale escrever a spec antes.
- Um sinal prático de que vibe coding passou do ponto: você está reexplicando a mesma restrição de negócio pela terceira vez numa sessão longa porque o agente "esqueceu" (ou o contexto foi comprimido, ver [Loop de tool use, contexto e permissions](../loop-de-tool-use-contexto-e-permissions/)) — nesse momento, capturar isso como spec formal em vez de repetir em prosa evita o problema se repetir.

## Onde isso converge com o "plan mode" de agentes de código

O modo de planejamento visto em harnesses modernos (planejar antes de editar, pedir aprovação do plano antes de tocar em código) é a versão leve/embutida de spec-driven development — em vez de um documento de spec externo mantido à parte, o próprio agente gera um plano estruturado como primeira etapa, você revisa e aprova, e só então a implementação começa. É o mesmo padrão de **planning explícito** já visto em [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/), aplicado especificamente ao contexto de escrever código: separar "decidir o que fazer" de "fazer", com um ponto de revisão humana entre os dois.

O próximo tópico, [Ferramentas de mercado](../ferramentas-de-mercado/), cobre as ferramentas específicas (spec-kit e afins) que formalizam esse fluxo além do que um plan mode embutido já oferece.
