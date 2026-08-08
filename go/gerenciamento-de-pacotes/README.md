# Gerenciamento de pacotes

## O problema que isso resolve

Quase todo programa real usa código escrito por outras pessoas — bibliotecas prontas para tarefas comuns (conectar num banco de dados, montar um servidor HTTP, gerar um UUID), em vez de reescrever tudo do zero. Isso levanta algumas perguntas que qualquer linguagem precisa resolver de algum jeito: como declarar quais bibliotecas externas o seu projeto usa? Como garantir que, ao rodar o projeto em outra máquina, seja baixada exatamente a mesma versão de cada dependência (e não uma versão mais nova que porventura mudou algo e quebrou seu código)? Como saber se uma dependência ficou obsoleta e não é mais usada por nada?

Em Go, esse sistema se chama **módulos** (o comando é `go mod`), e ele é parte da própria ferramenta `go`, sem precisar instalar nenhuma ferramenta externa para gerenciar dependências.

## go.mod — o manifesto do módulo

`go mod init <caminho>` (já visto de passagem no tópico [Sintaxe básica](../sintaxe-basica/#setup-iniciando-um-módulo)) cria um arquivo chamado `go.mod` na pasta do projeto. Esse arquivo é o "manifesto" do seu módulo: declara o nome dele, a versão mínima da linguagem Go exigida, e a lista de dependências diretas.

```
module github.com/gabriel/meu-projeto

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/google/uuid v1.3.0
)
```

Cada linha explicada:

- `module github.com/gabriel/meu-projeto` — declara o **caminho do módulo**. Isso não é só um nome de exibição: é literalmente o caminho que qualquer código (dentro do próprio projeto ou de outro projeto que venha a importar este) usa para importar pacotes daqui. Um pacote chamado `internal/user` dentro deste projeto seria importado como `import "github.com/gabriel/meu-projeto/internal/user"`. Por essa razão, renomear o caminho do módulo depois que outros projetos já dependem dele é uma mudança que quebra tudo que depende — não é uma decisão sem consequência.
- `go 1.22` — a versão mínima de Go que este projeto exige para compilar.
- `require (...)` — a lista de dependências diretas (bibliotecas externas que o código deste projeto importa de verdade), cada uma com a versão mínima aceita.

## go get — adicionar ou atualizar uma dependência

```
go get github.com/gin-gonic/gin@v1.9.1   # instala uma versão específica
go get github.com/gin-gonic/gin@latest   # instala a versão mais recente disponível
go get -u ./...                          # atualiza todas as dependências do projeto, respeitando versionamento semântico
```

Quando você roda `go get`, o comando baixa o código da dependência para um cache local (compartilhado entre todos os projetos Go da máquina, então cada versão de cada biblioteca só é baixada uma vez, não uma cópia por projeto) e, ao mesmo tempo, **escreve a versão resolvida diretamente no `go.mod`**. Não existe um estado intermediário de "baixei a dependência mas esqueci de declarar ela em algum arquivo" — o próprio comando que baixa já atualiza o manifesto do projeto.

## go.sum — não é o mesmo arquivo que o go.mod

Junto do `go.mod`, aparece automaticamente um segundo arquivo chamado `go.sum`, gerado pela própria ferramenta `go` (você nunca edita esse arquivo manualmente). Enquanto `go.mod` declara **quais** dependências diretas o projeto usa e a versão mínima aceita, `go.sum` guarda um **hash criptográfico** de cada versão de cada dependência já baixada — incluindo dependências indiretas (dependências das suas dependências, chamadas de "transitivas"):

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:...
```

Um hash criptográfico é como uma "impressão digital" do conteúdo de um arquivo — qualquer alteração, por menor que seja, no código baixado, produziria um hash diferente. Isso garante que, se em algum momento futuro alguém (ou algum ataque à cadeia de dependências) tentar substituir o código de uma versão específica por outro conteúdo com o mesmo número de versão, o build falha, porque o hash não bate mais com o que está registrado em `go.sum`.

`go.sum` **sempre deve ir para o controle de versão** (git), junto com `go.mod` — mesmo sendo um arquivo gerado automaticamente, ele nunca deve ser ignorado (não colocar no `.gitignore`), porque é exatamente ele que garante que qualquer pessoa que clonar o repositório vai baixar exatamente o mesmo código, byte a byte, que você usou.

## go mod tidy — sincroniza o que está declarado com o que é realmente usado

```
go mod tidy
```

Esse comando examina todo o código-fonte do projeto, procurando por instruções `import`, e faz duas coisas automaticamente:

1. Adiciona ao `go.mod` qualquer dependência que já está sendo importada em algum arquivo `.go`, mas que ainda não estava declarada.
2. Remove do `go.mod` qualquer dependência que está declarada, mas que nenhum arquivo `.go` do projeto importa mais (sobrou de um código que foi apagado, por exemplo).

O ponto importante: `go mod tidy` deriva o conteúdo do `go.mod` a partir do próprio código-fonte, automaticamente — é comum rodar esse comando depois de adicionar ou remover um `import` manualmente num arquivo, pra manter o `go.mod` sempre fiel ao que o código realmente usa, sem precisar editar o arquivo à mão.

## Versionamento semântico — e o caso especial da versão 2 em diante

Módulos Go seguem **versionamento semântico** (a convenção de numerar versões como `vMAJOR.MINOR.PATCH` — por exemplo `v1.9.1`) de forma obrigatória para a própria ferramenta funcionar direito, não como uma sugestão de estilo: a ideia central do versionamento semântico é que um incremento no número `MAJOR` (a primeira parte, de `v1` para `v2`) sinaliza uma mudança que quebra compatibilidade com código que já usava a versão anterior.

A partir da versão `v2` de um módulo, existe uma regra própria do sistema de módulos do Go: o **caminho de import precisa incluir o sufixo `/v2`**:

```go
import "github.com/gabriel/meu-projeto/v2/internal/user"
```

Essa regra existe justamente por causa da promessa do versionamento semântico: se `v2` pode ter uma API diferente de `v1` (afinal, é o que "quebra compatibilidade" quer dizer), então, do ponto de vista do sistema de import, `v1` e `v2` do mesmo módulo precisam ser tratados como pacotes tecnicamente distintos — e o sufixo no caminho de import é o que torna isso possível. Uma consequência prática interessante: um mesmo binário Go pode ter, ao mesmo tempo, uma dependência transitiva que usa a `v1` de um módulo e outra parte do código que usa a `v2` do mesmo módulo, sem conflito nenhum, porque para o sistema de módulos elas são caminhos de import diferentes.

## Vendoring — copiando o código das dependências para dentro do próprio repositório

```
go mod vendor           # cria uma pasta vendor/ com o código-fonte de cada dependência
go build -mod=vendor    # compila usando o código da pasta vendor/ em vez de ir buscar no cache
```

"Vendoring" é o nome dado à prática de copiar o **código-fonte completo** de cada dependência para dentro do próprio repositório do projeto (na pasta `vendor/`), em vez de depender só do cache local de módulos da máquina. Isso é útil em ambientes onde o build precisa funcionar sem acesso à rede (um ambiente de integração contínua isolado, por exemplo, ou uma exigência de auditoria/compliance corporativo que precisa que todo o código-fonte usado esteja versionado junto com o projeto). Não é o padrão do dia a dia hoje em dia (o cache de módulos padrão do Go já cobre a maior parte da necessidade de "build reprodutível sem depender de rede a cada vez"), mas ainda aparece em ambientes corporativos com essas restrições específicas.

## Módulos privados

```
export GOPRIVATE=github.com/minha-empresa/*
```

Por padrão, ao baixar uma dependência, `go get` tenta validar aquele código contra um serviço público (o "proxy" oficial de módulos Go, e um banco de dados público de checksums usado para verificação de integridade). Para código interno de uma empresa, que não deveria (e frequentemente nem consegue) ser validado contra um serviço público — validar assim tanto falharia quanto revelaria a existência de um repositório privado para um serviço externo —, a variável de ambiente `GOPRIVATE` avisa a ferramenta `go` para pular essas duas validações públicas nos caminhos que combinam com o padrão configurado, e ir buscar o código diretamente no sistema de controle de versão (autenticado, geralmente via SSH ou um arquivo de credenciais).

## Workspaces — trabalhando com vários módulos ao mesmo tempo, localmente

```
go work init ./api ./worker ./shared
```

Em um projeto grande, é comum dividir o código em vários módulos separados (cada um com seu próprio `go.mod`) — por exemplo, um módulo `api`, um módulo `worker` e um módulo `shared` que os dois anteriores usam. O problema: normalmente, para o módulo `api` enxergar uma mudança feita no módulo `shared`, seria preciso publicar uma nova versão de `shared` primeiro. Isso deixa o ciclo de "mudei o `shared`, quero testar já no `api`" bem mais lento durante o desenvolvimento local.

`go work init` cria um arquivo `go.work`, que faz a ferramenta `go` tratar os módulos listados como se as versões **locais no disco** de cada um fossem usadas entre si, sem precisar publicar nada — qualquer mudança feita em `shared` já fica visível imediatamente para `api` e `worker` durante o desenvolvimento. `go.work` normalmente **não vai para o controle de versão** (é configuração local de ambiente de desenvolvimento de cada pessoa), diferente de `go.mod`/`go.sum`, que sempre vão.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise gerenciamento de pacotes`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
