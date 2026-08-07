---
description: Gera um exercicio pratico avulso pra um topico (drill extra, fora do fluxo de aula completa do /topic)
argument-hint: [topico] [linguagem opcional — default go]
---

Gere um exercício prático avulso sobre: $ARGUMENTS (se a linguagem não for informada, assuma Go).

Diferença pro `/topic`: este comando não dá aula nem atualiza status em `*-MODULES.md` — é só pra praticar mais um tópico já visto (ou um tópico novo que o usuário quer treinar direto, sem passar pela explicação completa).

1. Se o tópico existir em `contexts/GO-MODULES.md` (ou outro `contexts/*-MODULES.md`), confira o campo "Contexto/notas" pra calibrar a dificuldade e evitar repetir erro já identificado antes.
2. Proponha um exercício curto, executável (`go run`/`go test` ou equivalente da linguagem), em nível adequado pra um engenheiro senior aprendendo a linguagem — sem explicação 101 do zero, direto ao cenário prático.
3. Quando fizer sentido, amarre o exercício a um princípio de `contexts/common/SOLID.md`/`contexts/common/CLEAN-CODE.md`, com exemplo em `config/reference/` da linguagem em questão.
4. Salve o código em `<linguagem>/<pasta-do-tópico>/exercise/` (crie a subpasta `exercise/` se não existir ainda); se o tópico não tiver pasta própria, use `infra/`/`ia/` quando o tema for especificamente disso, ou a raiz de `<linguagem>/` como fallback. `exercise/` é ignorada pelo git (ver `.gitignore`) — é rascunho de prática, não material de referência.
5. Se a pasta do tópico já tiver um `README.md` (criado pelo `/topic`), acrescente uma nota curta nele apontando pro exercício em `exercise/`; não recrie a explicação do zero. Se não tiver README nenhum ainda, não crie um só por causa do drill — isso é responsabilidade do `/topic`.
6. Não marque status em `*-MODULES.md` — esse controle é só do `/topic`.
