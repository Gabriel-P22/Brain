# Ponteiros, slices e maps

## Ponteiros — referência explícita

Em Python toda variável de objeto já é uma referência por baixo dos panos, de forma implícita. Go tem tipo valor de verdade (cópia ao passar/atribuir) e ponteiro explícito quando você quer referência:

```go
x := 10
p := &x     // p é *int, aponta pro endereço de x
*p = 20      // desreferencia e muda x através do ponteiro
fmt.Println(x) // 20
```

Sem aritmética de ponteiro (diferente de C) — não dá `p + 1` pra "andar" na memória. `&` pega endereço, `*` desreferencia; é isso.

Isso conecta direto com receiver por ponteiro do tópico anterior: passar um `*Order` pra uma função é o mesmo mecanismo — a função recebe o endereço, não uma cópia do struct.

## Arrays vs slices

Array tem tamanho fixo no tipo (`[5]int` é um tipo diferente de `[10]int`) — raramente usado direto. **Slice** é o que você usa no dia a dia, equivalente prático da `list` do Python:

```go
s := []int{1, 2, 3}       // slice literal
s = append(s, 4)           // slice cresce — mas atenção abaixo
sub := s[1:3]                // sublista, [2, 3] — igual list slicing do Python
```

Gotcha real (sem equivalente direto em Python): slice é uma "janela" sobre um array por baixo (ponteiro + tamanho + capacidade). Duas slices podem compartilhar o mesmo array:

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]      // b compartilha o array de a
b[0] = 99          // muda a[1] também! a agora é [1, 99, 3, 4, 5]
```

`append` só realoca (deixando de compartilhar) quando estoura a capacidade — então esse compartilhamento é "às vezes", o que é a fonte clássica de bug sutil em Go. Quando isolamento importa, copie explicitamente (`copy(dst, src)`).

## Maps — como dict, com duas pegadinhas

```go
m := map[string]int{"a": 1, "b": 2}
v, ok := m["c"]   // v = 0 (zero value de int), ok = false — Go não lança KeyError
```

Diferente do `dict[key]` do Python (que lança `KeyError`), acessar chave ausente em Go retorna o zero value do tipo — o idiom é sempre checar o segundo valor (`ok`) quando "existir ou não" importa.

Pegadinha 1 — **zero value de map é `nil`**, e escrever num map nil dá panic:

```go
var m map[string]int
m["a"] = 1   // panic: assignment to entry in nil map
```

Sempre inicialize com `make(map[K]V)` ou um literal `map[K]V{}` antes de escrever.

Pegadinha 2 — **iteração de map não tem ordem garantida** (de propósito — a linguagem randomiza pra ninguém depender disso). Diferente do `dict` do Python 3.7+, que preserva ordem de inserção. Se ordem importa, mantenha uma slice de chaves à parte.
