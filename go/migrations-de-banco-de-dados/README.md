# Migrations de banco de dados

## Por que migration versionada

Schema de banco muda junto com o código, e precisa mudar de forma rastreável e reproduzível — mesma motivação de qualquer linguagem (Alembic no Python/SQLAlchemy, Django migrations). Diferença é que Go não tem um ORM dominante com migration embutida por padrão (não existe um "Django" único) — a ferramenta de migration é escolhida à parte da lib de acesso a dado.

## Ferramentas comuns

- **golang-migrate**: CLI + lib, migrations em SQL puro (`000001_create_orders.up.sql` / `.down.sql`), agnóstico de framework.
- **goose**: parecido, com opção de escrever migration em Go quando SQL puro não é suficiente (ex: transformação de dado que precisa de lógica).
- **atlas**: mais recente, declarativo (você descreve o schema desejado, a ferramenta calcula o diff).

## Padrão up/down

```sql
-- 000001_create_orders.up.sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    amount NUMERIC NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 000001_create_orders.down.sql
DROP TABLE orders;
```

`up` aplica a mudança, `down` reverte — assim como em Alembic (`upgrade`/`downgrade`). Rodar via CLI (`migrate -path ./migrations -database $DSN up`) ou embutido no `main.go` do serviço (útil pra rodar migration automaticamente no deploy, com cuidado pra não rodar concorrente de múltiplas instâncias subindo ao mesmo tempo).

## Onde isso se conecta

Migration define a tabela; [Repository pattern na prática](../repository-pattern-na-pratica/) é o código Go que lê/escreve nela através da interface de domínio. Nunca gerar coluna nova só porque o struct Go mudou — schema é fonte da verdade explícita, versionada, não inferida do código (diferente de um ORM com `autogenerate` que às vezes mascara essa disciplina).
