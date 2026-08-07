# Tratamento de erros e pacotes

## error é só uma interface

Retomando o multi-retorno do tópico 1: `error` não é mágica de linguagem, é uma interface de um método:

```go
type error interface {
    Error() string
}
```

Qualquer tipo que tenha `Error() string` é um erro válido. Isso permite erro customizado com dado estruturado, algo que em Python normalmente exige subclasse de `Exception`:

```go
type NotFoundError struct{ ID string }
func (e *NotFoundError) Error() string { return fmt.Sprintf("id %s não encontrado", e.ID) }

return nil, &NotFoundError{ID: "123"}
```

## Wrapping — a versão Go do traceback

Python empilha traceback automaticamente quando uma exceção sobe. Go não empilha nada sozinho — se você não adicionar contexto, só sobra a mensagem original. O idiom é envolver o erro ao propagar:

```go
func loadUser(id string) (*User, error) {
    u, err := db.Find(id)
    if err != nil {
        return nil, fmt.Errorf("carregando usuário %s: %w", id, err)
    }
    return u, nil
}
```

`%w` (não `%v`) preserva o erro original encadeado — dá pra checar o tipo/valor original mesmo depois de embrulhado:

```go
var nf *NotFoundError
if errors.As(err, &nf) { /* trata especificamente */ }

if errors.Is(err, sql.ErrNoRows) { /* compara com sentinel error */ }
```

`errors.Is`/`errors.As` são o equivalente Go de `except SpecificError:` / `isinstance`, mas explícitos e chamados manualmente, não parte do controle de fluxo da linguagem.

## panic/recover não é try/except

`panic` existe, mas não é pra erro esperado (arquivo não encontrado, validação falhou) — isso sempre é `error` retornado. `panic` é pra estado realmente inconsistente (índice fora do slice, nil pointer dereference) — o equivalente mais próximo é um `AssertionError` do Python, não um `except` genérico. `recover()` (dentro de `defer`) existe pra não derrubar o processo inteiro, mas usar `recover` pra fluxo normal de erro é considerado anti-idiomático — vamos ver o uso legítimo dele (middleware HTTP) no Módulo 3.

## Pacotes

Pacote = pasta. Todo arquivo `.go` de uma pasta declara `package nome` no topo, e isso vira a unidade de organização — mais parecido com um módulo Python (`arquivo.py` = namespace) do que com pacote Python (pasta com `__init__.py`), mas obrigatório: não existe arquivo Go "solto" sem pacote.

Visibilidade é pela letra inicial do identificador — `PascalCase` exportado (visível fora do pacote), `camelCase` não. Sem `private`/`public` keyword, sem `__init__.py`, sem `_` de convenção só — é a própria linguagem que aplica.

Pacote pequeno e com um propósito claro (ex: `payment`, não `utils`) é Single Responsibility aplicado no nível de organização de código, não só de tipo — ver [contexts/common/SOLID.md](../../contexts/common/SOLID.md#s--single-responsibility-principle).
