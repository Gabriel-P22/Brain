---
description: Adiciona um novo topico - registra em contexts/*-MODULES.md e cria a pasta correspondente no projeto da linguagem
argument-hint: [linguagem] [nome-do-topico]
---

Adicione um novo tópico a partir de $ARGUMENTS (linguagem + nome do tópico).

1. Adicione a entrada `### Tópico: <nome>` em `contexts/<LINGUAGEM>-MODULES.md`, no módulo que fizer mais sentido (pergunte ao usuário qual módulo se não for óbvio pelo conteúdo) — mesmo formato dos tópicos existentes (`Status`, `Dia do plano`, `Conteúdo`, `Contexto/notas`).
2. Crie a pasta correspondente em `<linguagem>/<nome-do-topico-em-kebab-case>/`, já com a subpasta `exercise/` vazia dentro (mesmo padrão do `/init-lang`).
3. Se `<linguagem>/README.md` existir, atualize a lista de tópicos incluindo o novo na posição certa (dentro do módulo correto).
4. Não mexa em pastas de outros tópicos já existentes — só adiciona, não reorganiza o que já está lá.
