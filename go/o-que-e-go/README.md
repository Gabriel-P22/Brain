# O que é Go

## Origem e filosofia

Go (Golang) nasceu no Google em 2007, lançada publicamente em 2009 — criada por Robert Griesemer, Rob Pike e Ken Thompson (dois deles também por trás de Unix, C e UTF-8). Motivação real: compilação lenta e complexidade acumulada de C++ em bases de código gigantes do Google, sem abrir mão de performance.

Filosofia central é simplicidade radical: a especificação da linguagem é pequena (bem menos palavras-chave que Java/C++), existe um jeito idiomático "certo" pra cada coisa (reforçado por `gofmt`, que elimina debate de estilo por decreto), e compilação rápida mesmo em projetos grandes.

## Diferenças em relação a Python

- **Compilada, não interpretada**: `go build` gera binário nativo; produção não depende de runtime instalado (Python precisa do CPython no destino).
- **Tipagem estática**: erro de tipo pego em compile-time, não em runtime como Python.
- **Concorrência nativa na linguagem**: goroutines/channels são sintaxe da linguagem, não uma lib (`asyncio`) nem limitada por GIL (`threading`) — ver [concorrencia-goroutines-e-channels](../concorrencia-goroutines-e-channels/).
- **Sem herança de classe**: composição via embedding + interface implícita, não OOP clássica — ver [structs-metodos-e-interfaces](../structs-metodos-e-interfaces/).
- **Erro é valor, não exceção**: sem `try/except` — ver [tratamento-de-erros-e-pacotes](../tratamento-de-erros-e-pacotes/).
- **Deploy**: binário estático único; sem `venv`, sem casar versão de interpretador entre dev e produção.

## Como instalar

Neste ambiente já está instalado (via snap) — confirme com:

```
go version
```

Formas usuais de instalar: gerenciador do SO (`apt`, `brew`), download direto em go.dev/dl, ou uma ferramenta de versionamento tipo `asdf`/`mise` (equivalente ao `pyenv`) quando for preciso trocar de versão do Go por projeto.

`go env` mostra a configuração do ambiente (GOPATH, GOROOT etc.) — hoje, com Go Modules, isso importa bem menos no dia a dia do que importava há alguns anos.

## Diferenciais

- **Binário único estático**: `go build` produz um executável sem dependência externa — deploy é literalmente copiar um arquivo. Vantagem grande sobre Python, que precisa do interpretador certo + libs instaladas no destino.
- **Compilação rápida**: mesmo em bases de código grandes, diferencial forte frente a C++/Java.
- **Concorrência de primeira classe**: motivo nº 1 de Go ser escolha padrão em infra cloud-native — Docker, Kubernetes, Terraform e boa parte do ecossistema de infra moderna são escritos em Go.
- **Tooling embutido**: formatter (`gofmt`), testes (`go test`), profiler (`pprof`), race detector (`-race`) — sem precisar escolher/instalar ferramenta de terceiro pra nada disso.
- **Stdlib robusta pra rede/HTTP**: dá pra montar uma API só com `net/http`, sem framework — ver Módulo 3.

Contrapartida, pra não ficar parcial: Go é menos expressivo que Python pra prototipagem rápida (mais boilerplate, sem list comprehension, sem REPL forte), e o ecossistema de libs é menor que Python/JS em nichos específicos (ML/data science, por exemplo, onde Python domina sem disputa).
