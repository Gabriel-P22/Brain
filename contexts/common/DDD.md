# DDD (Domain-Driven Design)

Referência rápida e agnóstica de linguagem dos blocos táticos de DDD — pra citar sem reexplicar do zero. Exemplos concretos, no idioma de cada linguagem, ficam em `config/reference/` de cada linguagem: [go/config/reference/DDD.md](../../go/config/reference/DDD.md), [python/config/reference/DDD.md](../../python/config/reference/DDD.md) (e assim por diante conforme novas linguagens entrarem no vault).

DDD não é sobre framework ou ferramenta — é sobre modelar o código em cima da linguagem do negócio (o "domínio"), não em cima da estrutura de tabela do banco ou do endpoint HTTP.

## Entidade

Definida pela identidade (um ID), não pelos valores dos campos — dois objetos com mesmo conteúdo mas ID diferente são coisas diferentes; o mesmo objeto com campos mudados ao longo do tempo continua sendo "o mesmo". Regra de negócio da entidade deve morar em método da própria entidade, não espalhada num service genérico — entidade sem nenhuma regra própria (só getter/setter) é "anemic domain model", um anti-padrão.

## Value Object

Sem identidade própria — definido inteiramente pelo valor dos seus campos; dois Value Objects com mesmo valor são intercambiáveis. Imutável por convenção: operações retornam um novo Value Object em vez de mutar o existente.

## Agregado

Grupo de entidades/value objects que muda junto, como uma unidade de consistência, com uma **raiz** (aggregate root) como único ponto de entrada — nada de fora do agregado deve mutar uma entidade interna diretamente, só através da raiz. Só a raiz costuma ter repositório próprio; as entidades internas do agregado são carregadas/persistidas junto com ela.

## Repository

Abstração de persistência — interface que expõe operações de domínio (`Save`, `FindByID`), sem vazar detalhe de como o dado é armazenado. Pertence ao domínio (é declarado lá), não à infra — é Dependency Inversion aplicado (ver [SOLID.md](SOLID.md#d--dependency-inversion-principle)).

## Domain Service

Regra de negócio que envolve mais de uma entidade e por isso não pertence naturalmente a nenhuma delas sozinha (ex: transferir valor entre duas contas). Ainda é domínio puro — não depende de infra, só orquestra entidades/value objects.

## Bounded Context

Fronteira estratégica — dentro de um bounded context, um termo (ex: "Pedido") tem um significado único e consistente; em outro bounded context, o "mesmo" termo pode significar algo diferente (o "Pedido" do time de cobrança não é o "Pedido" do time de logística, mesmo com nome igual). Modelar isso como um tipo único compartilhado entre contextos diferentes é o erro estratégico mais comum — mistura o modelo mental de domínios distintos numa estrutura só.
