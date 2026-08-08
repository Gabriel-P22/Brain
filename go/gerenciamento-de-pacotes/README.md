# Gerenciamento de pacotes

## go.mod — o manifesto do módulo

`go mod init <caminho>` (já visto de passagem no Dia 1) cria o `go.mod`, equivalente ao `pyproject.toml`: nome do módulo, versão mínima da linguagem, dependências diretas.

```
module github.com/gabriel/meu-projeto

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/google/uuid v1.3.0
)
```

Diferença estrutural em relação a Python: o `module` path não é só um nome — é literalmente o caminho de import usado por quem consome esse código (`import "github.com/gabriel/meu-projeto/internal/user"`). Em Python o nome no `pyproject.toml` (`name = "meu-projeto"`) é só metadado de publicação; o import usa o nome do pacote local, sem relação obrigatória. Em Go, renomear o module path é um breaking change de verdade.

## go get — adicionar/atualizar dependência

```
go get github.com/gin-gonic/gin@v1.9.1   # versão específica
go get github.com/gin-gonic/gin@latest   # mais recente
go get -u ./...                          # atualiza tudo, respeitando semver
```

Equivalente a `pip install` ou `poetry add`, mas com uma diferença de fundo: `go get` sempre escreve a versão resolvida direto no `go.mod` — não existe "instalei mas esqueci de declarar" como pode acontecer com `pip install` sem atualizar o `requirements.txt` manualmente.

## go.sum — não é o go.mod

`go.mod` declara *quais* dependências (diretas) e a versão mínima aceita. `go.sum` é outro arquivo, gerado automaticamente, com o hash criptográfico de cada versão de cada dependência (direta e transitiva) já baixada:

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:...
```

É o equivalente mais próximo do `poetry.lock` (trava a árvore inteira, não só o topo) — bem mais forte que um `requirements.txt` sem hashes, que só trava versão sem garantir integridade do conteúdo baixado. `go.sum` **sempre vai pro git**, junto com `go.mod` — nunca é gitignored, mesmo sendo gerado.

## go mod tidy — sincroniza declarado com usado

```
go mod tidy
```

Escaneia o código, adiciona ao `go.mod` o que está importado e ainda não declarado, remove o que está declarado e não é mais usado por nada. Não tem equivalente direto e automático em `pip` — o mais parecido é rodar `pip-compile` sobre um `requirements.in`, mas ali quem decide o que entra ainda é você escrevendo à mão; o `go mod tidy` deriva isso do próprio código-fonte.

## Versionamento semântico — e o gotcha do `/v2`

Módulos Go seguem semver (`vMAJOR.MINOR.PATCH`) por convenção forte, não sugestão: o próprio toolchain assume que um bump de major é breaking change. A parte sem equivalente em Python: a partir de `v2`, o **caminho de import muda**, precisa do sufixo `/v2`:

```go
import "github.com/gabriel/meu-projeto/v2/internal/user"
```

Isso permite duas major versions do mesmo módulo coexistirem no mesmo binário como pacotes distintos — impossível de forma nativa em Python, onde `pip install pacote==1.0` e `pacote==2.0` simultâneos no mesmo ambiente exigem truque (namespacing manual, venvs separadas). Em Go isso é suportado pela própria resolução de módulos.

## Vendoring — dependência copiada pro repo

```
go mod vendor   # cria pasta vendor/ com o código-fonte de cada dependência
go build -mod=vendor
```

Mais próximo de commitar a `site-packages/` inteira do que de um `venv` — vendoring copia o **código-fonte** de cada dependência pra dentro do repo, pra builds que não podem depender de rede (CI isolado, compliance). Não é o padrão hoje (o cache de módulos do Go, `$GOPATH/pkg/mod`, já cobre a maior parte do caso de uso de "build reprodutível sem rede"), mas aparece em ambientes corporativos restritos.

## Módulos privados

```
export GOPRIVATE=github.com/minha-empresa/*
```

Sem isso, `go get` tenta validar o módulo contra o proxy público (`proxy.golang.org`) e o `sumdb` público — o que falha (ou vaza a existência do repo privado) pra código interno. `GOPRIVATE` avisa o toolchain pra pular proxy e checksum database nesses caminhos e ir direto no VCS (autenticado via `.netrc`/SSH, igual um `pip install git+ssh://...` particular). Equivalente ao índice privado de um `pip.conf`/`poetry` configurado com um `--extra-index-url` interno.

## Workspaces — múltiplos módulos, um único checkout local

```
go work init ./api ./worker ./shared
```

Cria um `go.work` que faz o toolchain tratar `./api`, `./worker` e `./shared` (cada um com seu próprio `go.mod`) como se as versões locais fossem usadas entre si, sem precisar publicar `shared` numa versão nova a cada mudança pra testar em `api`. É o equivalente Go de uma dependência `path = "../shared"` do Poetry, ou um monorepo com `pip install -e ../shared` — mas nativo do toolchain, sem editable install improvisado. `go.work` normalmente **não vai pro git** (é setup local de dev), diferente de `go.mod`/`go.sum`.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise gerenciamento de pacotes`; o código vai em `exercise/` (fora do git, ver `.gitignore`).
