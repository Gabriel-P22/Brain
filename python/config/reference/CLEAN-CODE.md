# Clean Code em Python

Como cada princípio (definição em [contexts/common/CLEAN-CODE.md](../../../contexts/common/CLEAN-CODE.md)) se aplica especificamente em Python — âncora de comparação ao estudar Go.

## Nomes

`snake_case` pra função/variável, `PascalCase` pra classe, `_prefixo` só convenção (não reforçada pelo interpretador) pra "privado". Diferente de Go, visibilidade não é imposta pela linguagem — é acordo de time.

## Funções

Retorno múltiplo via tupla é comum, mas menos "obrigatório" que em Go — muitas vezes um erro vira exceção em vez de valor de retorno. Parâmetro booleano isolado é o mesmo smell: geralmente duas funções disfarçadas de uma.

## Comentários

Docstring (`"""..."""`) é a convenção de documentação, lida por ferramentas (Sphinx, `help()`) — equivalente ao doc comment do Go, mas sem exigir letra maiúscula pra "ativar" a convenção.

## Formatação

`black`/`ruff format` cumprem o papel do `gofmt` — formatação automática, não debate manual. `ruff`/`flake8` cobrem lint.

## Tratamento de erros

Modelo diferente do Go: exceção, não valor de retorno. Regra de Clean Code aplicada aqui é não usar `except: pass` silencioso, e não capturar `Exception` genérica quando dá pra capturar o tipo específico — a "explicitação" que o Go força por tipo de retorno, em Python precisa de disciplina manual.

## Testes

`pytest` com `@pytest.mark.parametrize` é o equivalente do table-driven test do Go — uma função de teste, vários casos parametrizados.

## Code smells comuns

- **God class**: classe grande que devia ser 2-3 menores (ver SRP).
- **`except Exception:` genérico**: mascara erro que devia ser tratado especificamente ou propagado.
- **Mutable default argument** (`def f(x=[])`): armadilha específica de Python, sem equivalente direto em Go.
- **Getter/setter mecânico**: Python tem `@property` pra quando *precisa* de lógica no acesso — atributo público direto já resolve o caso simples, não criar `get_x()`/`set_x()` por hábito.
