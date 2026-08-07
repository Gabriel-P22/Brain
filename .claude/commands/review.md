---
description: Revisa o exercicio de Go mais recente (ou o arquivo indicado), aponta erros e sugere versao idiomatica
argument-hint: [caminho-do-arquivo (opcional)]
---

Se $ARGUMENTS apontar um arquivo, revise-o. Caso contrário, localize o exercício mais recente em `go/` (arquivo `.go` modificado por último).

Revise:
1. Corretude — o código compila e faz o que deveria (`go run`/`go test` quando aplicável).
2. Idiomaticidade Go — nomes curtos de variável, tratamento de erro explícito, uso correto de ponteiros/interfaces, evitar padrões "traduzidos" diretamente do Python que não fazem sentido em Go.
3. Compare com como um dev Python escreveria o equivalente, apontando a diferença de mentalidade quando relevante.

Aponte no máximo os problemas que importam para o aprendizado do dia — não sobrecarregue com nitpicks irrelevantes ao estágio atual do plano.
