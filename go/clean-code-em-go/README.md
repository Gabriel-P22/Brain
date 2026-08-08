# Clean Code em Go

## O que é Clean Code, resumidamente

"Clean Code" (código limpo) é um conjunto de práticas — não um framework nem uma ferramenta — pra escrever código que outra pessoa (ou você mesmo, seis meses depois) consegue ler e entender rápido, sem precisar de um manual explicando. A ideia central, detalhada em [contexts/common/CLEAN-CODE.md](../../contexts/common/CLEAN-CODE.md), gira em torno de nomes que revelam intenção, funções pequenas com um único nível de abstração, comentário só quando o código realmente não consegue se explicar sozinho, formatação automática (em vez de debate manual de estilo), erro tratado com contexto (nunca engolido em silêncio) e testes pequenos e independentes.

Este README mostra como cada uma dessas práticas se expressa na gramática específica de Go — algumas delas, em Go, deixam de ser "boa prática que depende de disciplina" e viram literalmente parte da ferramenta ou da própria linguagem.

## Ferramenta em vez de convenção

Boa parte do que em outros ambientes é convenção de equipe (formatação de código, organização de import) em Go é decisão automática de uma ferramenta que já vem junto com a instalação da linguagem:

- **`gofmt`**: formata qualquer arquivo `.go` no padrão oficial — indentação, espaçamento, onde quebrar linha. Não é sugestão, é decreto: todo código Go do mundo segue o mesmo padrão visual, porque a ferramenta decide, ninguém discute. Rodar `gofmt -l .` mostra quais arquivos ainda não estão formatados; `gofmt -w .` formata todos de uma vez.
- **`goimports`**: além de formatar como o `gofmt`, também organiza automaticamente a lista de `import` — remove import não usado (que em Go é, inclusive, **erro de compilação**, não só um aviso) e adiciona o que falta.
- **`golangci-lint`**: um "linter" — uma ferramenta que analisa o código procurando padrões arriscados ou pouco idiomáticos (variável declarada e nunca usada, erro ignorado sem querer, nome de função que não segue a convenção, etc.) e aponta cada ocorrência antes de o código ser revisado por uma pessoa.

Isso não elimina o julgamento humano — nome bom, função do tamanho certo, comentário no lugar certo continuam sendo decisão de quem escreve o código —, mas tira uma fatia inteira de "isso está formatado errado" ou "esse import está desorganizado" da revisão de código, porque a ferramenta já barra isso antes mesmo de chegar num Pull Request.

## Nomes

**Em palavras simples**: um nome bom permite entender o que uma variável, função ou tipo representa só de ler o nome, sem precisar ler o corpo inteiro pra descobrir. Escopo pequeno tolera (e em Go, até prefere) nome curto; escopo grande exige nome mais descritivo.

Uma particularidade importante de Go: **a letra inicial do nome não é só estilo — ela controla visibilidade**. Um identificador (nome de variável, função, tipo, campo de struct) que começa com letra maiúscula é **exportado**: visível e utilizável por qualquer outro pacote que importe este. Um identificador que começa com letra minúscula é **não exportado**: só visível dentro do próprio pacote onde foi declarado. Isso já foi visto em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) — aqui o ponto é que essa escolha de caixa (maiúscula/minúscula) é, ao mesmo tempo, uma decisão de design de API (o que devo deixar visível pro resto do programa?) e uma convenção de nome — as duas coisas estão fundidas na mesma decisão, o que é bem diferente da maioria das outras linguagens, onde visibilidade e nome são duas decisões separadas.

### Versão ruim

```go
package pedido

// Nome genérico demais — "d" não diz nada sobre o que representa.
// E "CalculaTudo" é vago: calcula o quê, exatamente?
func CalculaTudo(d []float64) float64 {
    t := 0.0
    for _, v := range d {
        t += v
    }
    return t
}
```

### Versão idiomática

```go
package pedido

// TotalItens soma o valor de cada item do pedido.
func TotalItens(valores []float64) float64 {
    total := 0.0
    for _, valor := range valores {
        total += valor
    }
    return total
}
```

Repare que dentro do próprio `for`, usar `valor` (em vez de algo mais longo tipo `valorDoItemAtual`) é perfeitamente idiomático — o escopo é de duas linhas, então um nome curto e claro já é suficiente. Variáveis de escopo bem pequeno e bem conhecido em Go (`i` num laço, `err` logo após uma chamada de função, `ctx` pra `context.Context`, `db` pra uma conexão de banco) são curtas por convenção da própria comunidade Go, não por preguiça de quem escreveu — o oposto (`indiceDoElementoAtual` num laço de 3 linhas) seria considerado ruído, não clareza.

## Funções

**Em palavras simples**: uma função deveria fazer uma coisa só, com um único nível de abstração (não misturar "passo bem específico e técnico" com "passo bem genérico e de alto nível" na mesma função), e ter poucos parâmetros. Um parâmetro booleano isolado (`func Processar(pedido Pedido, urgente bool)`) costuma ser sinal de que, na verdade, são duas funções disfarçadas de uma só — uma pra processamento normal, outra pra processamento urgente, cada uma com sua própria lógica.

Uma particularidade forte de Go: **funções costumam retornar mais de um valor**, e o padrão mais comum de todos é devolver "o resultado" junto com "o erro, se algo deu errado" (você já viu isso em [Sintaxe básica](../sintaxe-basica/)). Empacotar tudo num `struct` só pra "ter um retorno único" não é o idiomático em Go — múltiplos retornos fazem parte da gramática normal da linguagem. Se uma função está devolvendo mais de 2 ou 3 valores, aí sim é sinal de que provavelmente deveria ser um `struct`.

### Versão ruim

```go
// ProcessarPedido faz validação, cálculo de desconto E gravação —
// três níveis de abstração diferentes numa função só, e um parâmetro
// booleano isolado que na prática esconde dois comportamentos distintos.
func ProcessarPedido(p Pedido, aplicarDescontoVIP bool) error {
    if p.ID == "" {
        return errors.New("pedido sem ID")
    }
    if len(p.Itens) == 0 {
        return errors.New("pedido sem itens")
    }

    total := 0.0
    for _, item := range p.Itens {
        total += item.Valor
    }
    if aplicarDescontoVIP {
        total = total * 0.9
    }

    db, err := sql.Open("postgres", "...")
    if err != nil {
        return err
    }
    _, err = db.Exec("INSERT INTO pedidos (id, total) VALUES (?, ?)", p.ID, total)
    return err
}
```

### Versão refatorada

```go
func validar(p Pedido) error {
    if p.ID == "" {
        return errors.New("pedido sem ID")
    }
    if len(p.Itens) == 0 {
        return errors.New("pedido sem itens")
    }
    return nil
}

func calcularTotal(p Pedido) float64 {
    total := 0.0
    for _, item := range p.Itens {
        total += item.Valor
    }
    return total
}

// Em vez de um bool isolado, dois tipos de cliente distintos como
// política de desconto (o mesmo padrão já visto em SOLID em Go, seção O).
type PoliticaDesconto interface {
    Aplicar(valor float64) float64
}

func ProcessarPedido(p Pedido, desconto PoliticaDesconto, repo PedidoRepository) error {
    if err := validar(p); err != nil {
        return err
    }
    total := desconto.Aplicar(calcularTotal(p))
    return repo.Salvar(p.ID, total)
}
```

Cada função tem um único nível de abstração e um único motivo pra existir: `validar` só valida, `calcularTotal` só soma, e `ProcessarPedido` orquestra as outras num nível mais alto — sem descer aos detalhes de "como validar" ou "como calcular" dentro dela mesma. Repare que isso também é, na prática, o Single Responsibility Principle sendo aplicado (ver [SOLID em Go](../solid-em-go/)) — as duas disciplinas (Clean Code e SOLID) convergem pro mesmo tipo de código, mesmo vindo de origens diferentes: Clean Code é sobre legibilidade linha a linha, SOLID é sobre estrutura de dependência entre tipos.

## Comentários — e a exceção idiomática do godoc

**A regra geral de Clean Code**: comentário só deveria existir quando o código não consegue se explicar sozinho — um nome ruim não se resolve escrevendo um comentário do lado, se resolve renomeando a variável ou a função.

**Mas em Go existe uma exceção idiomática forte, que vale a pena aprender direito porque é diferente da regra geral**: todo identificador **exportado** (que começa com letra maiúscula — função, tipo, ou variável/constante visível fora do pacote) costuma levar um comentário de documentação logo acima dele, chamado de **doc comment**, seguindo um formato específico: a primeira palavra do comentário deve ser o próprio nome do identificador que está sendo documentado.

```go
// TotalItens soma o valor de cada item do pedido, ignorando itens
// com valor negativo.
func TotalItens(itens []Item) float64 {
    total := 0.0
    for _, item := range itens {
        if item.Valor < 0 {
            continue
        }
        total += item.Valor
    }
    return total
}

// Pedido representa um pedido de compra feito por um cliente.
type Pedido struct {
    ID     string
    Status Status
}
```

Esse comentário não é redundante nem "ruído" — ele alimenta uma ferramenta chamada `godoc`, que gera automaticamente uma documentação navegável do pacote a partir desses comentários (dá pra ver isso rodando `go doc nomedopacote` no terminal, ou olhando a documentação publicada de qualquer pacote conhecido em [pkg.go.dev](https://pkg.go.dev)). É por isso que essa é tratada como uma convenção **da própria linguagem**, diferente da regra geral de "evite comentário" — em Go, comentário em identificador exportado é esperado, não é considerado um cheiro de código.

Já um identificador **não exportado** (letra minúscula, só visível dentro do próprio pacote) segue a regra geral normal de Clean Code: comentário só se o nome e o código não conseguirem se explicar sozinhos.

```go
// Sem necessidade de comentário — o nome já diz tudo.
func calcularTotal(itens []Item) float64 {
    total := 0.0
    for _, item := range itens {
        total += item.Valor
    }
    return total
}
```

## Formatação

Como já visto acima: `gofmt` decide, ninguém discute. Não existe reunião de equipe pra decidir "chave na mesma linha ou não", "tab ou espaço", "quantas linhas em branco entre funções" — isso tira uma categoria inteira de debate improdutivo da revisão de código, deixando espaço pra revisão focar no que realmente importa (a lógica, os nomes, a estrutura).

## Tratamento de erros

Em Go, erro é um **valor de retorno normal**, não um mecanismo separado de "exceção" que interrompe o fluxo (isso foi visto em detalhe em [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/)). Isso tem uma consequência direta pra Clean Code: **ignorar um erro em Go é uma escolha visível**, não um acidente silencioso.

### Versão ruim

```go
func salvarArquivo(caminho string, dados []byte) {
    f, _ := os.Create(caminho)      // erro ignorado — e se o disco estiver cheio?
    f.Write(dados)                    // erro ignorado de novo
    f.Close()                          // e de novo
}
```

O `_` aqui está escondendo três chances diferentes de a operação falhar (permissão negada, disco cheio, caminho inválido) sem que nada avise ninguém — o programa simplesmente segue como se tivesse dado tudo certo.

### Versão correta

```go
func salvarArquivo(caminho string, dados []byte) error {
    f, err := os.Create(caminho)
    if err != nil {
        return fmt.Errorf("criando arquivo %s: %w", caminho, err)
    }
    defer f.Close()

    if _, err := f.Write(dados); err != nil {
        return fmt.Errorf("escrevendo em %s: %w", caminho, err)
    }
    return nil
}
```

Repare em dois detalhes idiomáticos importantes: (1) o erro é sempre propagado **com contexto** — `fmt.Errorf("criando arquivo %s: %w", caminho, err)` diz *onde* o erro aconteceu, em vez de simplesmente repassar `err` puro pra cima, o que deixaria quem lê o log adivinhando em qual etapa a falha ocorreu; (2) o `%w` (em vez de `%v` ou `%s`) preserva o erro original "embrulhado" dentro do novo, permitindo que quem recebe esse erro mais tarde use `errors.Is`/`errors.As` pra checar a causa raiz, mesmo depois de várias camadas de propagação.

O ponto de Clean Code aqui: em código com exceções tradicionais, um bloco de "capturar e ignorar" ainda compila e roda normalmente, e só aparece como problema quando o comportamento errado já causou um efeito colateral em produção — o código continua "parecendo limpo" na superfície. Em Go, um `_` no lugar de um erro é visualmente óbvio no meio do código e salta aos olhos de qualquer revisor treinado, exatamente porque o idiomático é sempre tratar ou propagar. Isso muda o comportamento de quem revisa código: "cadê o tratamento desse erro?" vira uma pergunta natural e imediata de revisão, porque a ausência do tratamento é visível, não escondida.

## Limites entre camadas (Boundaries)

Isolar uma dependência externa (uma biblioteca de terceiro, um SDK de algum serviço externo, uma API de LLM) atrás de uma interface própria — em vez de espalhar chamadas diretas a essa biblioteca pelo domínio inteiro — é uma prática de Clean Code que, na prática, é o mesmo Dependency Inversion Principle já visto em [SOLID em Go](../solid-em-go/) e em [Clean Architecture / separação em camadas](../clean-architecture-separacao-em-camadas/). Se amanhã a biblioteca externa mudar de versão (ou for trocada por outra), só o ponto de contato precisa mudar — não cada lugar do sistema que usava aquela biblioteca diretamente.

## Testes

O jeito idiomático de organizar teste em Go é o **table-driven test**: uma tabela (uma slice de `struct`) com os casos de teste, percorrida num `for`, onde cada linha da tabela representa logicamente "um teste", mesmo todas compartilhando a mesma função `Test...`.

```go
func TestTotalItens(t *testing.T) {
    casos := []struct {
        nome  string
        itens []Item
        want  float64
    }{
        {"lista vazia", []Item{}, 0},
        {"um item", []Item{{Valor: 10}}, 10},
        {"vários itens", []Item{{Valor: 10}, {Valor: 5}}, 15},
        {"ignora valor negativo", []Item{{Valor: 10}, {Valor: -3}}, 10},
    }

    for _, c := range casos {
        t.Run(c.nome, func(t *testing.T) {
            got := TotalItens(c.itens)
            if got != c.want {
                t.Errorf("TotalItens() = %v, want %v", got, c.want)
            }
        })
    }
}
```

`t.Run(c.nome, ...)` cria um subteste nomeado pra cada linha da tabela — se o caso `"ignora valor negativo"` falhar, o relatório de teste mostra exatamente esse nome, não só "TestTotalItens falhou" genérico. Isso mantém cada cenário curto, nomeado, e independente dos outros — exatamente os três critérios de teste limpo (um conceito por teste, nome que descreve o cenário, teste rápido e independente) citados em [contexts/common/CLEAN-CODE.md](../../contexts/common/CLEAN-CODE.md), só que organizados de um jeito específico da linguagem em vez de um teste separado por função pra cada caso.

## Code smells comuns em Go

- **`struct` "deus"**: um `struct` que deveria ser 2 ou 3 tipos menores, cada um com sua própria responsabilidade (ver Single Responsibility em [SOLID em Go](../solid-em-go/)).
- **Interface grande "pra já deixar pronto"**: desenhar uma interface com todos os métodos "que algum dia podem ser úteis", antes de existir um segundo consumidor real que precise dela. Go prefere a interface pequena, extraída só quando o segundo caso de uso real aparece — sem esse segundo consumidor, talvez nem precise virar interface ainda.
- **Erro ignorado (`_ = f()`)**: como visto acima, isso é ainda mais grave em Go do que pareceria à primeira vista, porque não existe nenhum mecanismo automático (tipo uma exceção subindo sozinha pela pilha de chamadas) que avise sobre o problema depois — se o erro foi descartado, ele simplesmente deixou de existir pro resto do programa.
- **Getter/setter mecânico sem nenhuma lógica**: em Go, um campo exportado (`PascalCase`, como `Pedido.Status`) já é diretamente acessível de fora do pacote — não há motivo pra criar `func (p Pedido) GetStatus() Status { return p.Status }` só por costume, como se fosse obrigatório encapsular todo acesso a campo atrás de um método. Só crie um método de acesso quando ele realmente faz algo além de devolver o valor cru (uma validação, um cálculo, uma conversão).
- **Abstração criada antecipadamente**: já mencionado acima como interface grande, mas vale generalizar — qualquer camada extra (um "wrapper" pra um wrapper, uma interface com uma única implementação e nenhuma alternativa em vista) criada só "porque parece mais profissional" é código a mais pra manter sem benefício real.

## Onde Clean Code encontra SOLID e DDD

Vale fechar amarrando os três: uma função pequena com um único nível de abstração (Clean Code) tende a já satisfazer Single Responsibility (SOLID) quase sem esforço extra — as duas disciplinas convergem pro mesmo resultado prático, mesmo vindo de origens diferentes (Clean Code nasce da preocupação com legibilidade linha a linha; SOLID nasce da preocupação com a estrutura de dependência entre tipos). Um sinal prático pra usar no dia a dia: se um método de uma entidade de domínio (ver [DDD em Go](../ddd-em-go/)) está difícil de nomear com um verbo curto e claro (`Cancelar`, `AdicionarItem`), geralmente é porque esse método está fazendo mais de uma coisa ao mesmo tempo — os dois problemas (nome ruim e responsabilidade múltipla) quase sempre aparecem juntos, e resolver um tende a resolver o outro.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise clean code em go`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
