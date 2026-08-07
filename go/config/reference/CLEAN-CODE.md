# Clean Code em Go

Como cada princípio (definição em [contexts/common/CLEAN-CODE.md](../../../contexts/common/CLEAN-CODE.md)) se aplica especificamente em Go.

## Nomes

Variável de escopo pequeno curta é idiomático (`i`, `err`, `ctx`, `db`), não preguiça. `camelCase` = não exportado, `PascalCase` = exportado — a caixa da primeira letra é parte do design (controla visibilidade), não é só estilo.

## Funções

Múltiplo retorno (`valor, err`) é o padrão — não empacotar tudo num `struct` só pra "ter um retorno único". Mais de 2-3 retornos é sinal de que devia ser `struct`.

## Comentários

Exceção idiomática forte: todo identificador exportado leva doc comment começando com o próprio nome (`// Order representa...`), pois alimenta o `godoc`. Isso é convenção da linguagem, não "comentário desnecessário" pela regra geral.

## Formatação

Não se discute: `gofmt` formata, `goimports` agrupa import, `golangci-lint` cobre boa parte do que em outras linguagens seria revisão manual.

## Tratamento de erros

Erro é valor de retorno — ignorá-lo (`_`) é escolha explícita e visível no code review. Propagar sempre com contexto: `fmt.Errorf("fazendo X: %w", err)`, nunca repassar nu sem dizer onde ocorreu.

## Testes

Idiomático: table-driven tests — uma tabela de casos (`struct{ name, input, want }`) percorrida num `for`, cada linha logicamente "um teste" mesmo compartilhando a função `Test...`.

## Code smells na transição de Python pra Go

- **God struct**: `struct` que devia ser 2-3 tipos menores.
- **Interface grande "pra já deixar pronto"**: Go prefere interface pequena descoberta a partir do uso real — sem 2º consumidor, talvez nem precise ser interface ainda.
- **Ignorar erro** (`_ = f()`): em Go isso é ainda mais grave que um `except: pass` em Python, porque não existe stack trace de exceção subindo sozinha depois.
- **Getter/setter mecânico**: campo exportado (`PascalCase`) já expõe o valor — não criar `GetX()`/`SetX()` como em Java/Python só por hábito, a menos que precise de lógica no acesso.
