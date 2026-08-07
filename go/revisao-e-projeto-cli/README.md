# Revisão e projeto CLI

Consolidação da Semana 1 — sem conceito novo, é aplicar tudo junto. Revisão rápida do que já foi coberto (detalhe em cada README):

- [Sintaxe básica](../sintaxe-basica/) — setup, variáveis, tipos, zero value, controle de fluxo, múltiplo retorno.
- [Structs, métodos e interfaces](../structs-metodos-e-interfaces/) — receiver, embedding, satisfação implícita de interface.
- [Ponteiros, slices e maps](../ponteiros-slices-e-maps/) — referência explícita, slice compartilhando array, map sem ordem garantida.
- [Tratamento de erros e pacotes](../tratamento-de-erros-e-pacotes/) — error como interface, wrapping com `%w`, visibilidade por caixa da letra.
- [Concorrência — goroutines e channels](../concorrencia-goroutines-e-channels/) — `go`, channel, goroutine leak.
- [Concorrência — sync, select e testes](../concorrencia-sync-select-e-testes/) — WaitGroup, Mutex, select, table-driven test.

## Projeto: CLI

Ideia pro projeto de consolidação: uma CLI pequena que bate tudo isso — ex. um "verificador de URLs" que recebe uma lista de endereços, checa cada um concorrentemente (goroutine + WaitGroup), agrega resultado (struct + slice), trata erro de rede sem derrubar o programa (error wrapping), e imprime relatório final.

Pacote `flag` da stdlib é o argparse do Go — mais limitado, mas cobre entrada de CLI simples sem dependência externa:

```go
url := flag.String("url", "", "URL a verificar")
flag.Parse()
```

Estrutura sugerida do pacote (`main` + um pacote interno pra lógica, já puxando um pouco de separação de camada do Módulo 2):

```
projeto-cli/
  main.go          // parsing de flag, orquestra
  checker/
    checker.go      // lógica pura, testável sem tocar em CLI/rede real
```

Esse projeto não tem exercício pré-gerado — quando quiser montar, `/exercise revisão e projeto cli` ou peça direto no chat.
