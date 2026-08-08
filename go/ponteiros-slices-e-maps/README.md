# Ponteiros, slices e maps

## Ponteiros — o endereço de uma variável na memória

Toda variável, quando o programa roda, fica guardada em algum lugar na memória do computador. Esse lugar tem um **endereço** — um número que identifica exatamente onde aquele dado está armazenado. Na maior parte do tempo você não precisa pensar nisso: você só usa o nome da variável (`x`, `idade`, `o`) e Go cuida de onde ela está guardada. Mas às vezes você precisa de acesso direto a esse endereço — e é isso que um **ponteiro** guarda.

```go
package main

import "fmt"

func main() {
    x := 10
    p := &x // & = "pega o endereço de": p agora guarda o endereço de memória onde x está

    fmt.Println(x)  // 10
    fmt.Println(p)  // algo como 0xc0000140a0 — um endereço de memória, não um número "normal"
    fmt.Println(*p) // 10 — * = "desreferencia": "vá até esse endereço e me dê o valor que está lá"
}
```

Dois símbolos, dois sentidos opostos:

- `&x` — "e comercial" na frente de uma variável: pega o endereço dela. O resultado é um ponteiro.
- `*p` — asterisco na frente de um ponteiro: desreferencia, ou seja, vai até aquele endereço e acessa o valor de verdade guardado lá.

O tipo de um ponteiro é escrito com asterisco antes do tipo apontado: `p` no exemplo acima é do tipo `*int` ("ponteiro para int"), porque `x` é `int`.

A parte que realmente importa na prática: através de um ponteiro, você consegue **alterar o valor original** de uma variável a partir de outro lugar do código — algo que não é possível se você só tem uma cópia do valor:

```go
package main

import "fmt"

func dobrar(n int) {
    n = n * 2 // muda só a cópia local — n aqui é um parâmetro separado de quem chamou
}

func dobrarComPonteiro(n *int) {
    *n = *n * 2 // desreferencia e muda o valor no endereço original
}

func main() {
    x := 10

    dobrar(x)
    fmt.Println(x) // 10 — não mudou, porque dobrar só recebeu uma CÓPIA de x

    dobrarComPonteiro(&x)
    fmt.Println(x) // 20 — mudou, porque dobrarComPonteiro recebeu o ENDEREÇO de x
}
```

Isso explica algo que já apareceu no tópico anterior sem esse nome: quando você declara um método com receiver por ponteiro (`func (o *Order) SetAmount(...)`, em [Structs, métodos e interfaces](../structs-metodos-e-interfaces/#receiver-por-valor-vs-por-ponteiro)), o mecanismo por baixo é exatamente este. Passar um `*Order` para uma função funciona pelo mesmo princípio de passar um `*int`: quem recebe o ponteiro consegue alterar o valor original, não uma cópia dele.

Importante: Go **não tem aritmética de ponteiro** (diferente de linguagens como C). Isso quer dizer que não existe operação tipo "pegue esse endereço e ande 4 posições na memória a partir dele" (`p + 1` não compila para ponteiros em Go). Os únicos dois usos possíveis de um ponteiro em Go são pegar o endereço (`&`) e desreferenciar (`*`) — isso elimina uma categoria inteira de bug clássico de baixo nível (acessar memória fora dos limites de uma variável por engano), na troca por um pouco menos de controle manual sobre memória.

Ponteiro também pode ser `nil` (lembrando o zero value do tópico [Sintaxe básica](../sintaxe-basica/#zero-value-toda-variável-já-nasce-com-um-valor-utilizável)) — quer dizer "não aponta para lugar nenhum ainda":

```go
var p *int   // p é nil — não aponta pra nenhum endereço
fmt.Println(p == nil) // true

*p = 10 // panic: runtime error: invalid memory address or nil pointer dereference
```

Tentar desreferenciar um ponteiro `nil` (isto é, tentar acessar o valor "no endereço" quando não existe endereço nenhum guardado) faz o programa quebrar imediatamente em tempo de execução — esse é provavelmente o erro mais comum que você vai encontrar enquanto aprende Go, então vale já saber reconhecer a mensagem.

## Arrays vs. slices — coleções de valores do mesmo tipo

Um **array** em Go tem tamanho fixo, e esse tamanho faz parte do próprio tipo — `[5]int` (array de 5 inteiros) e `[10]int` (array de 10 inteiros) são dois tipos diferentes, que não podem ser trocados um pelo outro. Por isso arrays são raramente usados diretamente no dia a dia:

```go
var a [3]int       // array de exatamente 3 inteiros, cada um começando em 0 (zero value)
a[0] = 10
a[1] = 20
fmt.Println(a)       // [10 20 0]
fmt.Println(len(a))  // 3 — tamanho fixo, sempre 3 pra esse array
```

O que você vai usar quase sempre, na prática, é o **slice**: uma coleção de tamanho variável, que consegue crescer conforme você adiciona itens.

```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3} // slice literal — repare que não tem número dentro dos colchetes, diferente de array
    fmt.Println(s)       // [1 2 3]
    fmt.Println(len(s))  // 3 — tamanho atual

    s = append(s, 4) // append devolve um NOVO slice — sempre reatribua o resultado
    fmt.Println(s)    // [1 2 3 4]

    sub := s[1:3] // fatia (slicing): do índice 1 até o 3, sem incluir o 3 — pega índices 1 e 2
    fmt.Println(sub) // [2 3]
}
```

Um detalhe de sintaxe que confunde no começo: `[]int{1, 2, 3}` é slice (colchetes vazios), enquanto `[3]int{1, 2, 3}` seria array (colchetes com o tamanho escrito). A diferença de um número dentro dos colchetes muda completamente o tipo.

`append` sempre **devolve** um slice novo — por isso o padrão é sempre reatribuir o resultado à mesma variável (`s = append(s, 4)`), nunca chamar `append(s, 4)` sozinho esperando que `s` mude por conta própria. Isso é consistente com o fato de Go sempre passar valores por cópia por padrão: `append` não tem como alterar a variável `s` de quem chamou sem que você reatribua o retorno dela.

### O gotcha real: slice é uma "janela" sobre um array por baixo

Este é o comportamento mais surpreendente de Go pra quem está começando, então vale um exemplo dedicado. Por baixo dos panos, um slice não é a coleção de dados em si — é uma estrutura pequena com três informações: um ponteiro para onde os dados realmente estão guardados (um array escondido), um tamanho atual (`len`) e uma capacidade máxima antes de precisar realocar (`cap`). Isso significa que **dois slices diferentes podem compartilhar o mesmo array por baixo**, e mudar um pode afetar o outro:

```go
package main

import "fmt"

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := a[1:3] // b "olha" para os índices 1 e 2 do MESMO array de a

    fmt.Println(b) // [2 3]

    b[0] = 99 // muda a posição 0 de b — que é a mesma memória da posição 1 de a
    fmt.Println(a) // [1 99 3 4 5] — a mudou também, mesmo sem tocar em "a" diretamente
    fmt.Println(b) // [99 3]
}
```

O que acontece: `a[1:3]` cria um slice `b` que não copia os dados — ele aponta para dentro do mesmo array que `a` já usa, só que começando na posição 1. Alterar `b[0]` está, na prática, alterando `a[1]`, porque são o mesmo espaço de memória visto por duas "janelas" diferentes.

Esse compartilhamento não é permanente, porém: quando um `append` precisa de mais espaço do que a capacidade atual permite, Go aloca um array novo, maior, copia os dados pra lá, e o slice resultante passa a apontar pra esse array novo — deixando de compartilhar com o slice original:

```go
c := make([]int, 3, 3) // slice com len=3 e cap=3 (sem espaço sobrando)
d := append(c, 4)        // estourou a capacidade — Go aloca um array NOVO pra d
d[0] = 999
fmt.Println(c) // c não mudou — d agora vive em outro array
```

Esse comportamento "às vezes compartilha, às vezes não" (depende só de ter estourado a capacidade ou não, algo que não é óbvio olhando o código) é a fonte mais comum de bug sutil envolvendo slices. Quando isolamento entre duas coleções realmente importa — por exemplo, ao devolver uma cópia de um slice interno para fora de uma função, pra garantir que quem recebe não consiga alterar o dado original por acidente — copie explicitamente com a função `copy`:

```go
original := []int{1, 2, 3}
copia := make([]int, len(original)) // cria um slice novo, vazio, do mesmo tamanho
copy(copia, original)                 // copia os valores de verdade, array separado

copia[0] = 999
fmt.Println(original) // [1 2 3] — intocado, porque copia vive num array totalmente separado
```

## Maps — coleções de chave e valor

Um **map** guarda pares de chave e valor, onde cada chave é única e serve pra buscar o valor associado a ela rapidamente:

```go
package main

import "fmt"

func main() {
    m := map[string]int{"a": 1, "b": 2} // chave string, valor int

    fmt.Println(m["a"]) // 1
    m["c"] = 3            // adiciona uma chave nova
    fmt.Println(m)         // map[a:1 b:2 c:3]

    v, ok := m["z"] // chave que não existe
    fmt.Println(v, ok) // 0 false
}
```

Repare no acesso a uma chave que não existe (`m["z"]`): Go **não interrompe o programa** nem gera um erro por isso — ele simplesmente devolve o zero value do tipo do valor (`0`, porque o map guarda `int`). Isso pode ser perigoso se você não checar, porque `0` pode ser um valor legítimo de verdade guardado no map, e você não teria como diferenciar "a chave existe e vale 0" de "a chave não existe". Por isso o idiomático em Go é sempre usar a forma de dois retornos quando "essa chave existe ou não" importa:

```go
v, ok := m["a"]
if ok {
    fmt.Println("encontrado:", v)
} else {
    fmt.Println("chave não existe")
}

// forma comum dentro de um if, usando a inicialização escoped (ver Sintaxe básica):
if v, ok := m["z"]; ok {
    fmt.Println("encontrado:", v)
} else {
    fmt.Println("não encontrado")
}
```

`ok` é `true` se a chave existe no map, `false` caso contrário — e `v` é o zero value quando `ok` é `false`.

### Pegadinha 1: o zero value de um map é `nil`, e escrever nele quebra o programa

```go
var m map[string]int // zero value de map é nil — não é um map "vazio utilizável", é a AUSÊNCIA de map
fmt.Println(m == nil) // true
fmt.Println(m["a"])   // 0 — LER de um map nil funciona, devolve zero value normalmente

m["a"] = 1 // panic: assignment to entry in nil map
```

Isso é diferente do zero value de outros tipos (como `int` virando `0`, pronto pra uso): o zero value de um map é `nil`, e um map `nil` só permite **leitura** (que sempre devolve zero value), não **escrita**. Antes de escrever em um map, sempre inicialize ele primeiro, de uma das duas formas:

```go
m1 := make(map[string]int)      // make cria um map vazio, pronto pra escrita
m1["a"] = 1                      // funciona

m2 := map[string]int{}          // literal vazio — equivalente a make aqui
m2["a"] = 1                      // funciona
```

### Pegadinha 2: iteração em um map não tem ordem garantida

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for chave, valor := range m {
    fmt.Println(chave, valor)
}
// a ordem em que "a", "b", "c" aparecem NÃO é garantida — e pode até mudar entre execuções diferentes do mesmo programa
```

Isso é uma decisão de design deliberada da linguagem: Go randomiza a ordem de iteração de um map de propósito, especificamente para impedir que qualquer código dependa acidentalmente de uma ordem específica (que, tecnicamente, nunca foi uma garantia de map nenhum, mesmo em outras linguagens onde a ordem costuma parecer estável na prática). Se a ordem de exibição importa para o seu programa, mantenha as chaves numa slice separada, ordenada do jeito que você precisa:

```go
import "sort"

chaves := make([]string, 0, len(m))
for k := range m {
    chaves = append(chaves, k)
}
sort.Strings(chaves) // agora chaves está em ordem alfabética

for _, k := range chaves {
    fmt.Println(k, m[k]) // itera m na ordem definida por "chaves", não na ordem "natural" do map
}
```

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise ponteiros, slices e maps`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
