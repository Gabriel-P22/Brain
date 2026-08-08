# Go

Estudo de Go pro cargo Go/Python + IA (APIs, infra, integração com LLM). Cronograma em [contexts/GO_STUDY_PLAN.md](../contexts/GO_STUDY_PLAN.md), conteúdo de cada tópico em [contexts/GO-MODULES.md](../contexts/GO-MODULES.md) — este README só lista a ordem de estudo e aponta pra pasta de cada tópico. Referência agnóstica (SOLID, Clean Code, DDD, Clean Architecture, boas práticas de backend) em [contexts/common/](../contexts/common/); exemplo idiomático de cada uma em Go, em [config/reference/](config/reference/).

Cada pasta de tópico tem seu próprio `README.md` (explicação, gerado por `/topic`) e uma subpasta `exercise/` (código de prática opcional, gerado por `/exercise`, fora do git).

## Módulo 1: Fundamentos

0. [O que é Go](o-que-e-go/)
1. [Sintaxe básica](sintaxe-basica/)
2. [Structs, métodos e interfaces](structs-metodos-e-interfaces/)
3. [Ponteiros, slices e maps](ponteiros-slices-e-maps/)
4. [Tratamento de erros e pacotes](tratamento-de-erros-e-pacotes/)
5. [Gerenciamento de pacotes](gerenciamento-de-pacotes/)
6. [Concorrência — goroutines e channels](concorrencia-goroutines-e-channels/)
7. [Concorrência — sync, select e testes](concorrencia-sync-select-e-testes/)
8. [Revisão e projeto CLI](revisao-e-projeto-cli/)

## Módulo 2: Boas práticas de engenharia

9. [SOLID em Go](solid-em-go/)
10. [DDD em Go](ddd-em-go/)
11. [Clean Architecture / separação em camadas](clean-architecture-separacao-em-camadas/)
12. [Clean Code em Go](clean-code-em-go/)

## Módulo 3: Backend em Go

Antes de começar, [contexts/common/BACKEND-BEST-PRACTICES.md](../contexts/common/BACKEND-BEST-PRACTICES.md) traz o pano de fundo agnóstico (design de API, erro/status code, observabilidade, config, segurança, resiliência, testes, contrato) que os tópicos abaixo aplicam concretamente em Go.

13. [net/http e APIs REST](net-http-e-apis-rest/)
14. [Gin (framework web)](gin-framework-web/)
15. [JSON e consumo de APIs externas](json-e-consumo-de-apis-externas/)
16. [Acesso a dados e context.Context](acesso-a-dados-e-context/)
17. [Migrations de banco de dados](migrations-de-banco-de-dados/)
18. [Repository pattern na prática](repository-pattern-na-pratica/)
19. [Tratamento de erros e exceções em APIs](tratamento-de-erros-e-excecoes-em-apis/)
20. [Middleware e chain de handlers](middleware-e-chain-de-handlers/)
21. [Validação de input](validacao-de-input/)
22. [Padrões de concorrência para infra](padroes-de-concorrencia-para-infra/)
23. [Produção — logging, config, error wrapping](producao-logging-config-error-wrapping/)
24. [Testes (unitários e integração)](testes-unitarios-e-integracao/)
25. [Projeto final — API + LLM](projeto-final-api-llm/)

## Módulo 4: Revisão final

26. [Revisão geral e boas práticas idiomáticas](revisao-geral-e-boas-praticas-idiomaticas/)
