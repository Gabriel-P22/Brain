# Migrations de banco de dados

## O que é "schema" e por que ele precisa de controle de versão

**Schema** é a estrutura de um banco de dados relacional: quais tabelas existem, quais colunas cada uma tem, os tipos de cada coluna, quais chaves e restrições existem. Assim como o código-fonte de um programa muda ao longo do tempo (uma feature nova, um campo a mais), o schema do banco também precisa mudar junto — uma tabela `orders` pode ganhar uma coluna `discount_code` quando o sistema passa a suportar cupom de desconto, por exemplo.

O problema: se cada pessoa do time altera o schema manualmente, direto no banco (rodando um `ALTER TABLE` avulso quando lembra), rapidamente surgem problemas sérios:

- O banco de um desenvolvedor fica com um schema diferente do banco de outro colega, porque um rodou o `ALTER TABLE` manualmente e o outro esqueceu.
- Ninguém sabe, olhando o banco de produção, exatamente **quais** mudanças de schema já foram aplicadas e em que ordem.
- Não existe um jeito confiável de reproduzir o schema do zero (por exemplo, para subir um ambiente de teste novo) sem lembrar manualmente de cada mudança feita ao longo de meses.

Uma **migration** ("migração", no sentido de "migrar o schema de uma versão para a próxima") resolve isso: cada mudança de schema vira um **arquivo versionado, guardado no controle de versão junto com o código** (ex: git), com um número de ordem. Uma ferramenta de migration aplica esses arquivos, um de cada vez, em ordem, e guarda dentro do próprio banco de dados um registro de quais migrations já foram aplicadas — assim, rodar a ferramenta de novo só aplica o que ainda falta, nunca repete uma migration já aplicada.

## Ferramentas de migration em Go

Go não tem um framework de acesso a dado dominante com sistema de migration embutido por padrão — a ferramenta de migration é escolhida separadamente da biblioteca de acesso ao banco (`database/sql`, ver [Acesso a dados e context.Context](../acesso-a-dados-e-context/)). As opções mais comuns no ecossistema:

- **[golang-migrate](https://github.com/golang-migrate/migrate)** — CLI e biblioteca, migrations escritas em SQL puro, um par de arquivos por mudança (`.up.sql` para aplicar, `.down.sql` para reverter). Agnóstico de framework, funciona com vários bancos diferentes.
- **[goose](https://github.com/pressly/goose)** — proposta parecida (CLI + biblioteca), com a vantagem extra de permitir escrever uma migration **em código Go** quando SQL puro não é suficiente — por exemplo, quando a mudança de schema exige transformar dados existentes com lógica que SQL puro descreveria de forma difícil de ler.
- **[atlas](https://atlasgo.io)** — mais recente, com uma abordagem **declarativa**: em vez de você escrever "o que fazer para chegar lá" (como nas duas ferramentas acima), você descreve **o schema desejado final**, e a ferramenta calcula automaticamente o diff entre o schema atual e o desejado, gerando a migration necessária.

Este tópico foca em golang-migrate (mais comum em times que já têm SQL bem definido) e goose (quando é útil ter a opção de escrever uma migration em Go).

## O padrão up/down

Cada mudança de schema vira dois arquivos: um que **aplica** a mudança (`up`) e um que **reverte** ela (`down`) — dá para "andar para frente" e "andar para trás" no histórico de schema, algo essencial quando um deploy precisa ser revertido rapidamente:

```sql
-- 000001_create_orders.up.sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    amount NUMERIC NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- 000001_create_orders.down.sql
DROP TABLE orders;
```

```sql
-- 000002_add_discount_code.up.sql
ALTER TABLE orders ADD COLUMN discount_code TEXT;
```

```sql
-- 000002_add_discount_code.down.sql
ALTER TABLE orders DROP COLUMN discount_code;
```

O prefixo numérico (`000001`, `000002`) garante a ordem de aplicação — golang-migrate também aceita prefixo baseado em timestamp (`20260101120000_create_orders.up.sql`), útil em times grandes onde duas pessoas podem criar uma migration no mesmo dia e um número sequencial manual geraria conflito.

## Rodando golang-migrate

Via linha de comando, depois de instalado (`go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest`):

```
migrate -path ./migrations -database "$DATABASE_URL" up      # aplica todas as migrations pendentes
migrate -path ./migrations -database "$DATABASE_URL" up 1    # aplica só a próxima migration pendente
migrate -path ./migrations -database "$DATABASE_URL" down 1  # reverte só a última migration aplicada
migrate -path ./migrations -database "$DATABASE_URL" version # mostra a versão atual aplicada no banco
```

Também dá para usar como biblioteca, embutido no próprio binário Go (útil para rodar a migration automaticamente no momento do deploy):

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func aplicarMigrations(databaseURL string) error {
    m, err := migrate.New("file://migrations", databaseURL)
    if err != nil {
        return fmt.Errorf("carregando migrations: %w", err)
    }
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        // ErrNoChange não é um erro de verdade — só significa "já estava tudo aplicado"
        return fmt.Errorf("aplicando migrations: %w", err)
    }
    return nil
}
```

Repare nos dois imports em branco (`_`) — o mesmo mecanismo de registro de driver visto em [Acesso a dados e context.Context](../acesso-a-dados-e-context/), aqui usado para registrar o suporte a Postgres e a "fonte" de migrations vindas de arquivo local.

## goose: quando SQL puro não basta

goose segue o mesmo padrão de arquivos numerados, mas com uma marcação de comentário especial que separa a parte `up` da parte `down` dentro do **mesmo** arquivo:

```sql
-- +goose Up
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    amount NUMERIC NOT NULL
);

-- +goose Down
DROP TABLE orders;
```

Quando a mudança de dado precisa de lógica que SQL puro deixaria difícil de ler ou manter (por exemplo, migrar dados de um formato de coluna antigo para um novo, com regras de transformação específicas), goose permite escrever a migration inteira em Go:

```go
package migrations

import (
    "database/sql"

    "github.com/pressly/goose/v3"
)

func init() {
    goose.AddMigration(upNormalizeEmails, downNormalizeEmails)
}

func upNormalizeEmails(tx *sql.Tx) error {
    rows, err := tx.Query("SELECT id, email FROM users")
    if err != nil {
        return err
    }
    defer rows.Close()

    for rows.Next() {
        var id, email string
        if err := rows.Scan(&id, &email); err != nil {
            return err
        }
        normalizado := strings.ToLower(strings.TrimSpace(email))
        if _, err := tx.Exec("UPDATE users SET email = $1 WHERE id = $2", normalizado, id); err != nil {
            return err
        }
    }
    return rows.Err()
}

func downNormalizeEmails(tx *sql.Tx) error {
    return nil // essa transformação não tem um "desfazer" sensato — down vazio é aceitável aqui
}
```

Essa migration roda dentro de uma transação (`*sql.Tx`) — se qualquer linha falhar no meio do processo, o `goose` desfaz tudo automaticamente, em vez de deixar o banco num estado parcialmente migrado.

## Rodando migration automaticamente no deploy — com cuidado

É comum embutir a chamada de `aplicarMigrations` (como no exemplo acima) no início do `main.go` do serviço, para que o schema seja atualizado automaticamente a cada deploy, sem passo manual. Isso exige um cuidado real: se **múltiplas instâncias do serviço** sobem ao mesmo tempo (comum em deploy com vários pods/containers), todas elas tentariam rodar a mesma migration simultaneamente, o que pode causar erro de concorrência no schema (duas instâncias tentando criar a mesma tabela ao mesmo tempo, por exemplo). A forma padrão de evitar isso é usar um **lock consultivo** (advisory lock) do próprio banco de dados durante a aplicação da migration — tanto golang-migrate quanto goose já cuidam disso automaticamente ao rodar migrations em Postgres, mas vale confirmar esse comportamento antes de confiar cegamente nisso em produção com múltiplas instâncias.

## Onde isso se conecta com o resto do curso

Migration define **a tabela**; [Repository pattern na prática](../repository-pattern-na-pratica/) é o código Go que lê e escreve nela através de uma interface de domínio. Uma regra importante para manter essa relação saudável: **nunca crie uma coluna nova só porque um struct Go mudou** — o schema do banco é a fonte da verdade explícita e versionada, e o struct Go é quem deve refletir o schema, não o contrário. Isso é bem diferente do comportamento de um ORM com geração automática de migration a partir do modelo de código (`autogenerate`), que às vezes mascara essa disciplina, gerando migrations sem uma revisão cuidadosa de que exatamente aquela mudança de schema é a intenção real.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise migrations de banco de dados`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: instalar golang-migrate, criar duas migrations (`create_orders` e `add_discount_code`), aplicar as duas, depois reverter uma com `down 1` e confirmar que a coluna `discount_code` sumiu.
