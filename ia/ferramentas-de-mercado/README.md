# Ferramentas de mercado

Continuação de [Spec-driven vs vibe coding](../spec-driven-vs-vibe-coding/) — ali ficou o conceito; aqui é o panorama de ferramentas que formalizam esse fluxo, e como isso se relaciona com o que você já usa no dia a dia.

## Panorama: do plan mode embutido a ferramentas dedicadas

Existe um espectro de formalização, do mais leve ao mais estruturado:

1. **Plan mode embutido num harness** (ex: no Claude Code) — o agente gera um plano antes de editar, você aprova, ele executa. Vive inteiramente dentro da sessão de chat, sem artefato persistente fora dela por padrão.
2. **Templates/skills que estruturam a spec** — convenções ou automações (como as usadas neste próprio vault, ver `.claude/commands/` e `contexts/plans/`) que fazem o agente escrever a spec/plano como um documento salvo em arquivo antes de implementar, criando rastreabilidade sem depender de uma ferramenta externa.
3. **Ferramentas dedicadas de spec-driven development** — projetos como o **spec-kit** (open-source, iniciado pela GitHub) definem um fluxo formal e reutilizável entre múltiplos agentes/harnesses: `/specify` (escrever a especificação do que construir), `/plan` (detalhar a abordagem técnica), `/tasks` (quebrar em tarefas executáveis), com cada etapa gerando um artefato revisável antes da próxima. A motivação declarada desses projetos é justamente reduzir a dependência de "vibe coding" em bases de código maiores, tornando a intenção auditável em cada etapa, não só no resultado final.

O ganho de uma ferramenta dedicada sobre plan mode embutido é reusabilidade e portabilidade — a spec vira um artefato em arquivo, versionado no repositório, que sobrevive à sessão de chat e pode ser retomado por outra pessoa (ou outro agente) sem depender de reconstruir o contexto do zero.

## Como isso converge com prática de engenharia já conhecida

Nada disso é conceitualmente novo para quem já passou por processo de RFC, design doc ou requirements doc formal antes de um projeto grande — spec-kit e ferramentas parecidas só automatizam o "próximo passo" (implementação) a partir de um artefato que times de engenharia madura já produziam manualmente. A diferença prática: antes, escrever a spec era trabalho de um humano e implementar também; agora, escrever a spec continua sendo trabalho principalmente humano (com ajuda do agente para rascunhar), mas implementar a partir dela pode ser majoritariamente delegado — o que muda o cálculo de quando vale a pena formalizar (ver trade-off de [Spec-driven vs vibe coding](../spec-driven-vs-vibe-coding/)).

## Neste vault

Este próprio repositório já usa uma versão leve desse padrão: `.claude/commands/` define fluxos reutilizáveis (`/topic`, `/exercise`, `/init-lang`) que geram artefatos persistentes (READMEs de tópico, `*-MODULES.md`) em vez de deixar o conteúdo só na conversa — e `contexts/plans/` guarda planos salvos de sessões anteriores. É a mesma motivação de spec-driven development (intenção e progresso sobrevivendo além do chat) aplicada à escala de um vault de estudo, não de um projeto de produção.

## Cuidado ao avaliar ferramenta nova

Esse espaço muda rápido — ferramentas específicas (nomes, features exatas) evoluem e surgem com frequência. O critério que não muda para avaliar qualquer ferramenta nova nesse espaço: ela reduz a distância entre "o que foi pedido" e "o que foi implementado" de forma auditável, ou só adiciona cerimônia sem gerar um artefato de valor real? A mesma pergunta que já se faz para qualquer processo/ferramenta de engenharia nova, aplicada aqui.
