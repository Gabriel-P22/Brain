---
description: Cria a estrutura de uma nova linguagem no vault (pasta, config/reference/ com SOLID.md e CLEAN-CODE.md idiomaticos — so exemplo, sem redefinir principio —, e opcionalmente um *-MODULES.md)
argument-hint: [nome-da-linguagem]
---

Crie a estrutura pra estudar uma nova linguagem: $ARGUMENTS.

1. Crie a pasta `<linguagem>/` na raiz do vault (nome em minúsculo, ex: `rust/`, `typescript/`).
2. Crie `<linguagem>/config/reference/SOLID.md` e `<linguagem>/config/reference/CLEAN-CODE.md`, seguindo exatamente a estrutura de `go/config/reference/SOLID.md` e `go/config/reference/CLEAN-CODE.md` (leia esses dois como modelo) — mesmos títulos de seção, só exemplo de código idiomático da nova linguagem (nunca "traduzido" de Go ou Python, e sem redefinir o princípio — a explicação de cada letra/conceito vive só em contexts/). Linke de volta pra definição agnóstica em `../../../contexts/common/SOLID.md` e `../../../contexts/common/CLEAN-CODE.md`.
3. Adicione essa linguagem nos links de "exemplos concretos ficam em..." dentro de `contexts/common/SOLID.md` e `contexts/common/CLEAN-CODE.md`.
4. Pergunte ao usuário se ele quer também um `contexts/<LINGUAGEM>-MODULES.md` (lista de tópicos por módulo, no mesmo formato de `contexts/GO-MODULES.md`) — só crie se ele confirmar, já que nem toda linguagem nova precisa de plano de estudo formal (pode ser só material de referência).
5. Atualize a seção "Folder layout" do CLAUDE.md com a nova pasta.
