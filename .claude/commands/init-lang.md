---
description: Inicializa a estrutura de topicos de uma linguagem (README + uma pasta por topico, na ordem de estudo) a partir do *-MODULES.md em contexts/
argument-hint: [linguagem]
---

Inicialize a pasta de estudo de $ARGUMENTS (ex: `go`, `python`).

1. Confira se `<linguagem>/` já tem alguma pasta de tópico (qualquer pasta que não seja `config/`). Se já tiver, pare e pergunte antes de mexer — não sobrescreva progresso existente.
2. Localize `contexts/<LINGUAGEM>-MODULES.md` (ex: `contexts/GO-MODULES.md` pra `go`). Se não existir, avise o usuário e sugira rodar `/new-language` primeiro, ou pergunte se ele quer seguir só com o README, sem pastas de tópico.
3. Crie `<linguagem>/README.md` com: visão geral rápida da linguagem nesse vault, link pra `contexts/<LINGUAGEM>-MODULES.md` e pro `*_STUDY_PLAN.md` correspondente (quando existir), e a lista de tópicos na ordem de estudo (módulo por módulo), cada um linkando pra sua pasta.
4. Crie uma pasta por tópico dentro de `<linguagem>/`, nome em kebab-case a partir do nome do tópico (sem prefixo numérico — a ordem de estudo fica só no README; isso evita ter que renomear pasta quando um tópico novo entrar no meio da lista depois via `/add-topic`). Dentro de cada pasta de tópico, crie também uma subpasta `exercise/` (vazia, fica no `.gitignore` — é onde o `/exercise` salva o rascunho de prática quando gerado, separado do `README.md` de explicação que o `/topic` escreve).
5. Não mexa em `<linguagem>/config/` — é material de referência (SOLID/Clean Code, configs de tooling), não tópico de estudo.
