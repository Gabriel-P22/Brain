# DDD em Go

Definição de cada bloco tático: [contexts/common/DDD.md](../../contexts/common/DDD.md). Exemplo de código por bloco: [go/config/reference/DDD.md](../config/reference/DDD.md). Este README não repete isso — é sobre o que muda especificamente em Go.

## O que Go não te dá de graça (e Python/Java às vezes dão)

Sem ORM ativo estilo Django (onde o model já é a entidade e a persistência ao mesmo tempo) — em Go, entidade de domínio e struct de persistência costumam ser dois tipos diferentes, convertidos explicitamente na implementação do repository. Isso é mais boilerplate, mas é exatamente o que mantém o domínio sem import de banco (Clean Architecture, ver [clean-architecture-separacao-em-camadas](../clean-architecture-separacao-em-camadas/)).

Sem encapsulamento de campo forçado pela linguagem (Go não tem `private` de verdade dentro do mesmo pacote) — a fronteira do agregado (só a raiz é mutável de fora) é convenção de design + visibilidade de pacote (não exportar construtor de `OrderItem` fora do pacote `order`, por exemplo), não uma trava do compilador como seria um campo `private` em Java.

## Erro comum vindo de Python

Modelar entidade como "bag de dados" com toda regra num `service.py`/`service.go` à parte (anemic domain model) — hábito reforçado em Python por frameworks que incentivam separar `models.py` "burro" de `views.py`/`services.py` "inteligente". Em DDD (e isso vale igual nas duas linguagens), a regra que pertence à entidade deve morar no método da entidade — `Order.Cancel()`, não `OrderService.CancelOrder(order)` reimplementando a regra por fora.

## Quando NÃO aplicar DDD tático completo

Nem todo pacote pequeno precisa de Entity + Value Object + Aggregate + Repository + Domain Service formalizados — pra um CRUD simples sem regra de negócio real, isso é over-engineering (viola a mesma lógica do "não force padrão em exercício pequeno" já visto em [SOLID em Go](../solid-em-go/)). DDD tático compensa quando existe complexidade de regra de negócio genuína pra proteger — não pela complexidade em si.
