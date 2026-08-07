# Clean Architecture

Referência rápida e agnóstica de linguagem — pra citar sem reexplicar do zero. Exemplos concretos (estrutura de pasta/pacote real), no idioma de cada linguagem, ficam em `config/reference/` de cada linguagem: [go/config/reference/CLEAN-ARCHITECTURE.md](../../go/config/reference/CLEAN-ARCHITECTURE.md), [python/config/reference/CLEAN-ARCHITECTURE.md](../../python/config/reference/CLEAN-ARCHITECTURE.md) (e assim por diante conforme novas linguagens entrarem no vault).

O mesmo espírito aparece com nomes diferentes (Clean Architecture, Hexagonal/Ports & Adapters, Onion Architecture, "camadas" em DDD) — a ideia central é sempre a mesma regra.

## A regra de dependência

Dependência de código aponta sempre pra dentro, do detalhe pro domínio — nunca o contrário. Domínio não conhece banco, framework web, biblioteca externa nenhuma.

## As camadas

- **Domínio**: regra de negócio pura — entidades, value objects, regra de domínio. Não importa nada de fora.
- **Aplicação / caso de uso**: orquestra o domínio pra realizar uma operação completa (ex: "criar pedido"), ainda sem saber qual banco/framework está por trás.
- **Infraestrutura**: implementação concreta de tudo que o domínio/aplicação declararam como abstração — banco, fila, API externa, cache.
- **Interface / entrega (delivery)**: como o mundo externo aciona o caso de uso — HTTP, CLI, gRPC, mensageria. Deveria ser trocável sem alterar domínio nem aplicação.

## Teste prático: "dá pra testar isso isolado?"

Se a regra de negócio só roda quando o banco está de pé (ou o servidor HTTP subiu), a regra de dependência não está sendo respeitada — o teste mais rápido de "a arquitetura está limpa" é conseguir testar o domínio sem infra nenhuma no ar.

## Por que vale desenhar isso desde cedo

Em código dinâmico é barato misturar camadas no início e desembaraçar depois. Em código onde retrofit de abstração é mais caro (compilado, tipado, sem "monkey patch"), misturar camadas cedo custa caro depois — não significa over-engineerizar um exercício pequeno, significa que, num serviço real, vale desenhar a fronteira desde o começo.
