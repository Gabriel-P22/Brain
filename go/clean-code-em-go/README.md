# Clean Code em Go

Definição de cada princípio: [contexts/common/CLEAN-CODE.md](../../contexts/common/CLEAN-CODE.md). Exemplo/convenção por princípio: [go/config/reference/CLEAN-CODE.md](../config/reference/CLEAN-CODE.md) (nomes, funções, doc comment/godoc, `gofmt`, erro, testes, code smells). Este README não repete isso — é a síntese de por que Go "força" mais Clean Code do que Python força.

## Ferramenta em vez de convenção

Boa parte do que em Python é convenção de time (formatação, algumas regras de nome, import organizado) em Go é decisão automática de tooling: `gofmt` formata sem debate, `goimports` organiza import, `golangci-lint` cobre boa parte do que seria comentário de PR manual em outras linguagens. Isso não elimina julgamento — nome bom, função pequena, comentário no lugar certo continuam sendo decisão humana — mas tira uma fatia inteira de "code smell" da revisão de código porque a ferramenta já barra antes de chegar no PR.

## A maior diferença prática: erro forçado a ser visível

Em Python, `except: pass` é code smell mas compila, roda, e some silenciosamente até dar problema em produção. Em Go, ignorar erro (`_ = f()`) é sintaticamente possível mas visualmente óbvio — no diff, no code review, `_` salta aos olhos de um jeito que um `except` vazio não salta. Isso muda o comportamento de quem revisa código: em Go, "cadê o tratamento desse erro?" é uma pergunta natural de review porque a ausência é visível.

## Onde Clean Code encontra SOLID/DDD aqui

Função pequena com um nível de abstração (Clean Code) tende a já satisfazer Single Responsibility (SOLID) sem esforço extra — as duas disciplinas convergem pro mesmo tipo de código na prática, mesmo vindo de origens diferentes (Clean Code é sobre legibilidade linha a linha, SOLID é sobre estrutura de dependência). Um sinal prático: se um método de uma entidade DDD (ver [DDD em Go](../ddd-em-go/)) está difícil de nomear num verbo curto e claro, geralmente ele está fazendo mais de uma coisa — os dois problemas (nome ruim, responsabilidade múltipla) costumam aparecer juntos.
