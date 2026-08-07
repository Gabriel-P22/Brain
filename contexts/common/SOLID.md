# SOLID

Referência rápida e agnóstica de linguagem dos 5 princípios — pra citar sem reexplicar do zero. Exemplos concretos, sempre no idioma da própria linguagem (nada de código "traduzido" de outra), ficam em `config/reference/` de cada linguagem: [go/config/reference/SOLID.md](../../go/config/reference/SOLID.md), [python/config/reference/SOLID.md](../../python/config/reference/SOLID.md) (e assim por diante conforme novas linguagens entrarem no vault).

## S — Single Responsibility Principle

Uma unidade de código deve ter um único motivo pra mudar. Se uma classe/tipo/módulo mistura validação + persistência + notificação, são 3 motivos de mudança — três responsabilidades que deveriam estar em três lugares.

## O — Open/Closed Principle

Aberto pra extensão, fechado pra modificação. Adicionar um novo comportamento não deve exigir editar o código existente que já funciona — só adicionar uma nova implementação/unidade ao lado. Um `switch`/`if-elif` gigante sobre "tipo" que cresce a cada feature nova é o sinal clássico de violação.

## L — Liskov Substitution Principle

Qualquer implementação de um contrato (interface/classe base) deve poder substituir outra implementação do mesmo contrato sem quebrar o comportamento esperado por quem consome. Vale tanto pra tipo/assinatura quanto pra comportamento — uma implementação que lança erro onde outras não lançam, ou ignora parte do contrato implícito, quebra LSP mesmo quando o compilador/interpretador não reclama.

## I — Interface Segregation Principle

Preferir vários contratos pequenos e específicos a um grande genérico — ninguém deveria ser forçado a depender de método que não usa. Contrato grande com muitos métodos, onde cada consumidor só usa uma fração deles, é sinal de quebrar em contratos menores.

## D — Dependency Inversion Principle

Módulos de alto nível não devem depender de módulos de baixo nível; ambos devem depender de abstrações. A abstração (interface/contrato) deveria "pertencer" a quem consome, não a quem implementa — isso é o que permite trocar a implementação concreta (banco, fila, API externa) sem tocar na regra de negócio.

---

Ao estudar um tópico novo, sempre se guiar pelo modelo idiomático da linguagem em questão (ver arquivo da linguagem) — a forma como cada princípio se expressa muda bastante entre linguagens com herança de classe e linguagens com composição/interface implícita.
