# O que é Go

## Em uma frase

Go (também chamada de **Golang**) é uma linguagem de programação criada para escrever programas simples, rápidos de rodar e fáceis de manter — usada principalmente para construir os bastidores de aplicações: servidores, APIs, ferramentas de linha de comando e infraestrutura (o tipo de software que fica rodando sem interface visual, atendendo pedidos ou processando dados).

## Quem criou e por quê

Go nasceu dentro do Google em 2007 e foi lançada publicamente em 2009. Foi criada por três pessoas — Robert Griesemer, Rob Pike e Ken Thompson — sendo que os dois últimos também ajudaram a criar o sistema operacional Unix e a linguagem C, décadas antes.

O motivo de criar Go foi bem prático: o Google tinha bases de código gigantescas (milhões de linhas), escritas principalmente em C++, e via dois problemas graves:

1. **Compilar demorava demais.** "Compilar" é o processo de transformar o código que você escreve em um programa que o computador consegue executar. Em bases de código grandes de C++, esse processo podia levar minutos ou até dezenas de minutos, cada vez que alguém queria testar uma mudança pequena.
2. **A linguagem tinha ficado complexa demais.** Muitos recursos, muitas formas diferentes de resolver o mesmo problema, muita coisa pra decorar antes de conseguir ler o código de outra pessoa com confiança.

Go foi desenhada pra resolver exatamente isso: compilar rápido mesmo em projetos enormes, e ter uma gramática pequena o suficiente pra caber na cabeça de qualquer desenvolvedor sem precisar de anos de experiência.

## Simplicidade como filosofia central

Isso não é só discurso de marketing — dá pra ver isso em decisões concretas da linguagem:

- **Poucas palavras-chave.** Go tem 25 palavras reservadas no total (`if`, `for`, `func`, `return`, etc.). Para comparação de escala: isso é bem menos do que a maioria das linguagens populares usadas em produção.
- **Um jeito "certo" de formatar código, e ponto.** Go vem com uma ferramenta chamada `gofmt` que formata automaticamente qualquer arquivo `.go` no padrão oficial da linguagem — indentação, espaçamento, onde quebrar linha. Isso elimina uma categoria inteira de discussão de equipe ("tab ou espaço?", "chave na mesma linha ou não?") porque a ferramenta decide por decreto, e todo código Go do mundo segue o mesmo padrão visual.
- **Sem excesso de formas de fazer a mesma coisa.** Go evita deliberadamente ter múltiplos jeitos "elegantes" de escrever a mesma lógica. Menos flexibilidade de estilo, mas também menos decisão de design que cada desenvolvedor precisa tomar sozinho.

## Compilada, não interpretada — o que isso significa na prática

Linguagens de programação se dividem, de forma simplificada, em duas famílias quanto a como o código vira algo que o computador executa:

- **Linguagens compiladas** (Go, C, Rust): existe uma etapa explícita, antes de rodar o programa, que lê todo o código-fonte e gera um **binário** — um arquivo executável nativo, no formato que o processador do computador entende diretamente. Depois de gerado, esse binário roda sozinho, sem precisar do código-fonte nem de nenhuma ferramenta extra instalada na máquina onde ele vai rodar.
- **Linguagens interpretadas**: o código-fonte é lido e executado linha a linha, em tempo real, por um programa chamado interpretador, que precisa estar instalado na máquina que vai rodar o programa.

Em Go, o comando `go build` faz essa compilação: ele lê seus arquivos `.go` e gera um único arquivo binário. Esse arquivo pode ser copiado para outro computador (mesmo sistema operacional/arquitetura) e executado direto, sem precisar instalar Go nele, sem precisar de nenhuma outra dependência de sistema. Isso simplifica muito o processo de colocar um programa em produção: "fazer deploy" de um programa em Go, no caso mais simples, é literalmente copiar um arquivo.

## Tipagem estática — o que isso significa na prática

"Tipo" é a categoria de um dado: número inteiro, texto, verdadeiro/falso, etc. "Tipagem estática" quer dizer que **o tipo de cada variável é fixado e checado antes do programa rodar** (durante a compilação), não descoberto durante a execução.

```go
var idade int = 25
idade = "vinte e cinco" // ERRO DE COMPILAÇÃO — nem chega a virar binário
```

O compilador rejeita esse código antes mesmo de tentar rodá-lo, porque `idade` foi declarada como `int` (número inteiro) e o código tenta colocar nela um texto (`string`). Esse tipo de erro — misturar tipos por engano — é pego cedo, no momento de compilar, em vez de aparecer como um bug silencioso só quando aquele trecho específico do programa rodar em produção.

Isso tem um custo: você precisa ser mais explícito sobre o tipo de cada coisa (embora Go tenha inferência de tipo em vários casos, como você vai ver no próximo tópico). E tem um ganho grande: uma categoria inteira de erros comuns simplesmente não compila, então nunca chega a acontecer com um usuário real.

## Concorrência nativa — o maior diferencial de Go

"Concorrência" é a capacidade de um programa lidar com várias tarefas ao mesmo tempo — por exemplo, um servidor atendendo centenas de pedidos de clientes diferentes simultaneamente, sem que um pedido lento trave todos os outros.

Isso é tão importante em Go que a linguagem tem uma sintaxe própria, embutida na gramática da linguagem, só para isso: a palavra-chave `go` na frente de uma chamada de função dispara essa função para rodar de forma concorrente com o resto do programa:

```go
go fazerAlgoDemorado() // dispara e continua a execução sem esperar terminar
```

Essas unidades de execução concorrente em Go se chamam **goroutines**, e são muito mais leves (em termos de memória e custo de troca de contexto) do que as "threads" tradicionais do sistema operacional — um programa Go comum consegue rodar milhares delas ao mesmo tempo sem problema. O tópico [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) entra em detalhe em como isso funciona. Por enquanto, o que importa saber é: isso é o motivo número 1 pelo qual Go virou a linguagem padrão de facto em infraestrutura de nuvem — Docker, Kubernetes, Terraform e boa parte do ecossistema moderno de infra são escritos em Go.

## Erro como valor, não como exceção

Muitas linguagens têm um mecanismo de "exceção": quando algo dá errado, o programa "levanta" um erro que interrompe o fluxo normal e precisa ser "capturado" em algum bloco especial (frequentemente chamado de `try`/`catch` ou `try`/`except`) em algum ponto acima na cadeia de chamadas.

Go não tem esse mecanismo para erros esperados do dia a dia. Em vez disso, uma função que pode falhar simplesmente **retorna um erro como um valor comum**, junto com o resultado:

```go
resultado, err := dividir(10, 0)
if err != nil {
    // trate o erro aqui — é obrigatório olhar pra essa variável
    fmt.Println("deu erro:", err)
    return
}
fmt.Println("resultado:", resultado)
```

Isso é explicado a fundo no tópico [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/). Por enquanto, o ponto central é: em Go, tratar erro é uma parte visível e explícita da leitura do código, não algo que acontece "escondido" em algum lugar distante de onde o problema realmente ocorreu.

## Como instalar

Neste ambiente Go já está instalado. Para confirmar a versão instalada:

```
go version
```

Formas usuais de instalar em uma máquina nova:
- Gerenciador de pacotes do sistema operacional (`apt`, `brew`, etc.)
- Download direto em [go.dev/dl](https://go.dev/dl)
- Uma ferramenta de gerenciamento de versões como `asdf` ou `mise`, útil quando você precisa alternar entre versões diferentes do Go em projetos diferentes na mesma máquina

O comando `go env` mostra a configuração do ambiente Go instalado (variáveis como `GOPATH`, `GOROOT`). Hoje em dia, com o sistema de módulos do Go (`go.mod` — ver [Gerenciamento de pacotes](../gerenciamento-de-pacotes/)), essas variáveis importam bem menos no dia a dia do que importavam há alguns anos.

## Seu primeiro programa em Go

Todo programa Go executável começa com essa estrutura mínima:

```go
package main

import "fmt"

func main() {
    fmt.Println("Olá, Go!")
}
```

Explicando cada linha:

- `package main` — todo arquivo `.go` pertence a um **pacote** (uma forma de agrupar código relacionado). O pacote chamado `main` é especial: é o que diz ao Go "isso aqui é um programa executável", não uma biblioteca para outros programas importarem.
- `import "fmt"` — traz para este arquivo o pacote `fmt` da biblioteca padrão do Go (a "standard library", que já vem instalada junto com a linguagem, sem precisar baixar nada). `fmt` tem funções para formatar e imprimir texto.
- `func main()` — a função chamada `main`, dentro do pacote `main`, é o **ponto de entrada** do programa: é a primeira coisa que roda quando você executa o binário.
- `fmt.Println("Olá, Go!")` — chama a função `Println` do pacote `fmt`, que imprime o texto no terminal seguido de uma quebra de linha.

Para rodar esse arquivo sem precisar compilar manualmente primeiro:

```
go run arquivo.go
```

`go run` compila e executa em um único passo — ótimo enquanto você está experimentando. Para gerar o binário de verdade (o que você usaria em produção):

```
go build arquivo.go
./arquivo   # no Linux/Mac
```

Não existe um "modo interativo" padrão embutido no Go (nenhum terminal onde você digita uma linha de código e vê o resultado na hora) — todo código roda a partir de um arquivo.

## Diferenciais de Go, resumidos

- **Binário único estático**: `go build` produz um único arquivo executável, sem dependência externa — copiar esse arquivo para o servidor de produção já é o deploy.
- **Compilação rápida**: mesmo em bases de código com milhões de linhas, compilar em Go continua rápido — foi literalmente o problema original que a linguagem foi criada para resolver.
- **Concorrência de primeira classe**: goroutines e channels são sintaxe da própria linguagem, não uma biblioteca externa — o principal motivo de Go dominar em ferramentas de infraestrutura moderna.
- **Ferramentas já inclusas**: formatador de código (`gofmt`), executor de testes (`go test`), analisador de performance (`pprof`), detector de condição de corrida em código concorrente (`-race`) — tudo isso já vem com a instalação do Go, sem precisar escolher nem instalar nenhuma ferramenta de terceiros.
- **Biblioteca padrão forte para rede e web**: dá para montar uma API HTTP funcional usando só o pacote `net/http`, que já vem embutido — sem precisar de nenhum framework externo (o Módulo 3 deste curso mostra isso na prática).

## Contrapartida, para não ficar parcial

Go não é a ferramenta certa para tudo. Por ser uma linguagem deliberadamente simples e explícita, ela tende a exigir mais código para tarefas de prototipagem rápida e exploração de dados do que linguagens focadas nesse tipo de uso — não existe atalho de sintaxe para transformar/filtrar listas em uma linha, por exemplo, nem um modo interativo forte para testar ideias rapidamente. E em áreas muito específicas — como ciência de dados e aprendizado de máquina — o ecossistema de bibliotecas prontas em Go é bem menor do que em linguagens que dominam esses nichos. Go compensa isso sendo excelente exatamente no que ela foi desenhada para ser boa: serviços de backend, APIs e infraestrutura que precisam ser simples, rápidos e confiáveis em produção.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise o que é go`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: digitar o "Olá, Go!" acima em um arquivo `main.go`, rodar com `go run main.go`, depois gerar o binário com `go build` e rodá-lo diretamente.
