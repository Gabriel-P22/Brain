---
description: Carrega o contexto de um topico em GO-MODULES.md (ou outro *-MODULES.md) e inicia a aula/exercicio
argument-hint: [nome-ou-trecho-do-topico]
---

Localize em `contexts/GO-MODULES.md` (ou em outro `contexts/*-MODULES.md` do vault, se $ARGUMENTS indicar outro assunto — python, arquitetura, IA) a seção `### Tópico:` cujo nome corresponde a $ARGUMENTS.

1. Leia o campo "Contexto/notas" do tópico para saber o que já foi coberto antes, evitando repetir explicações.
2. Explique o conteúdo do tópico comparando com o equivalente (ou ausência de equivalente) em Python, já que esse é o modelo mental de referência do usuário.
3. Proponha um exercício prático curto e executável (`go run`/`go test`) para fixar o conteúdo, salvando o código na pasta apropriada (`go/`, `infra/` ou `ia/`, dependendo do tópico).
4. Ao final, marque o tópico como concluído (`Status: [x]`) no `contexts/*-MODULES.md`, atualize "Contexto/notas" com um resumo do que foi visto e eventuais dificuldades do usuário, e — se o tópico tiver um "Dia do plano" associado — marque esse dia como concluído (`- [x]`) em `contexts/GO_STUDY_PLAN.md` também.
