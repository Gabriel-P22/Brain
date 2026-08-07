# Go

Estudo de Go pro cargo Go/Python + IA (APIs, infra, integração com LLM). Cronograma em [contexts/GO_STUDY_PLAN.md](../contexts/GO_STUDY_PLAN.md), conteúdo de cada tópico em [contexts/GO-MODULES.md](../contexts/GO-MODULES.md) — este README só lista a ordem de estudo e aponta pra pasta de cada tópico. Referência agnóstica (SOLID, Clean Code, DDD, Clean Architecture, boas práticas de backend) em [contexts/common/](../contexts/common/); exemplo idiomático de cada uma em Go, em [config/reference/](config/reference/).

Cada pasta de tópico tem seu próprio `README.md` (explicação, gerado por `/topic`) e uma subpasta `exercise/` (código de prática opcional, gerado por `/exercise`, fora do git).

## Módulo 1: Fundamentos

0. [O que é Go](o-que-e-go/)
1. [Sintaxe básica](sintaxe-basica/)
2. [Structs, métodos e interfaces](structs-metodos-e-interfaces/)
3. [Ponteiros, slices e maps](ponteiros-slices-e-maps/)
4. [Tratamento de erros e pacotes](tratamento-de-erros-e-pacotes/)
5. [Concorrência — goroutines e channels](concorrencia-goroutines-e-channels/)
6. [Concorrência — sync, select e testes](concorrencia-sync-select-e-testes/)
7. [Revisão e projeto CLI](revisao-e-projeto-cli/)

## Módulo 2: Boas práticas de engenharia

8. [SOLID em Go](solid-em-go/)
9. [DDD em Go](ddd-em-go/)
10. [Clean Architecture / separação em camadas](clean-architecture-separacao-em-camadas/)
11. [Clean Code em Go](clean-code-em-go/)

## Módulo 3: Backend em Go

Antes de começar, [contexts/common/BACKEND-BEST-PRACTICES.md](../contexts/common/BACKEND-BEST-PRACTICES.md) traz o pano de fundo agnóstico (design de API, erro/status code, observabilidade, config, segurança, resiliência, testes, contrato) que os tópicos abaixo aplicam concretamente em Go.

12. [net/http e APIs REST](net-http-e-apis-rest/)
13. [Gin (framework web)](gin-framework-web/)
14. [JSON e consumo de APIs externas](json-e-consumo-de-apis-externas/)
15. [Acesso a dados e context.Context](acesso-a-dados-e-context/)
16. [Migrations de banco de dados](migrations-de-banco-de-dados/)
17. [Repository pattern na prática](repository-pattern-na-pratica/)
18. [Tratamento de erros e exceções em APIs](tratamento-de-erros-e-excecoes-em-apis/)
19. [Middleware e chain de handlers](middleware-e-chain-de-handlers/)
20. [Validação de input](validacao-de-input/)
21. [Padrões de concorrência para infra](padroes-de-concorrencia-para-infra/)
22. [Produção — logging, config, error wrapping](producao-logging-config-error-wrapping/)
23. [Testes (unitários e integração)](testes-unitarios-e-integracao/)
24. [Projeto final — API + LLM](projeto-final-api-llm/)

## Módulo 4: Revisão final

25. [Revisão geral e boas práticas idiomáticas](revisao-geral-e-boas-praticas-idiomaticas/)
