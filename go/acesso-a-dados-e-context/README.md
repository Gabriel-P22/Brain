# Acesso a dados e context.Context

## O que é um banco de dados relacional, rapidamente

Um **banco de dados** é um programa separado (roda como seu próprio processo, geralmente em outra máquina ou container) especializado em guardar dados de forma organizada e duradoura — diferente da memória de um programa, que some quando o programa termina. Um banco de dados **relacional** (Postgres, MySQL, SQLite, entre outros) organiza esses dados em **tabelas** (linhas e colunas, parecido com uma planilha) e é consultado através de uma linguagem chamada **SQL** (Structured Query Language) — comandos de texto como `SELECT`, `INSERT`, `UPDATE`, `DELETE`.

O seu programa Go, para conversar com um banco desses, precisa abrir uma **conexão de rede** com ele (o banco quase sempre está rodando em outro processo, escutando numa porta, do mesmo jeito que um servidor HTTP escuta requisições — ver [net/http e APIs REST](../net-http-e-apis-rest/)) e mandar comandos SQL por essa conexão, recebendo os resultados de volta.

## O pacote database/sql e o driver

A biblioteca padrão do Go tem um pacote chamado `database/sql` que define uma **interface genérica** para conversar com qualquer banco relacional — mas esse pacote sozinho não sabe falar o protocolo de rede específico de nenhum banco. Quem sabe fazer isso é o **driver**: uma biblioteca de terceiros específica para cada banco (para Postgres, exemplos comuns são `lib/pq` ou `pgx`; para MySQL, `go-sql-driver/mysql`).

O driver se registra no `database/sql` através de um import especial, chamado **import em branco** (`_`):

```go
import (
    "database/sql"

    _ "github.com/lib/pq" // import só pelo efeito colateral de registrar o driver "postgres"
)
```

O `_` na frente do caminho de import diz ao Go "importe este pacote e rode o código de inicialização dele, mas eu não vou usar nenhum identificador dele diretamente no meu código" — sem o `_`, o compilador reclamaria de "importado e não usado" (erro de compilação em Go, não um aviso — ver [Sintaxe básica](../sintaxe-basica/)). O pacote do driver, ao ser carregado, se registra internamente com um nome (`"postgres"`, no caso do `lib/pq`) que você usa depois para abrir a conexão.

## Abrindo a conexão: sql.Open é preguiçoso (lazy)

```go
db, err := sql.Open("postgres", dsn) // "postgres" é o nome do driver registrado; dsn é a string de conexão
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

Um detalhe importante: `sql.Open` **não conecta de fato ao banco** nessa chamada. Ele só valida o formato da string de conexão (`dsn` — Data Source Name, algo como `"postgres://usuario:senha@localhost:5432/meubanco"`) e prepara a estrutura interna (`*sql.DB`) que vai gerenciar as conexões depois. A conexão de rede real só acontece na primeira vez que você executa uma query. Isso significa que `sql.Open` quase nunca retorna erro por "banco fora do ar" — para checar se o banco está realmente acessível, use `db.Ping()` (ou `db.PingContext(ctx)`) explicitamente, geralmente no momento de subir a aplicação.

## Pool de conexões: por que ele existe

Abrir uma conexão de rede nova com um banco de dados é uma operação relativamente cara — envolve handshake de rede, autenticação, alocação de recursos no lado do banco. Se cada requisição HTTP que chega no seu servidor abrisse (e fechasse) uma conexão nova com o banco, o custo dessa abertura/fechamento repetida dominaria o tempo de resposta.

Um **pool de conexões** resolve isso mantendo um conjunto de conexões já abertas e prontas para reuso: quando seu código pede uma query, o pool empresta uma conexão livre do conjunto (ou abre uma nova, se precisar e ainda houver espaço); quando a query termina, a conexão volta para o pool em vez de ser fechada, pronta para a próxima query que vier.

No Go, esse pool já vem embutido dentro do `*sql.DB` — não é algo que você precisa montar nem instalar separadamente. Mas os parâmetros do pool valem a pena configurar explicitamente, porque os padrões nem sempre combinam com o limite real do seu banco em produção:

```go
db.SetMaxOpenConns(25)                  // no máximo 25 conexões abertas ao mesmo tempo, contando em uso + ociosas
db.SetMaxIdleConns(5)                   // no máximo 5 conexões ociosas guardadas prontas pra reuso
db.SetConnMaxLifetime(30 * time.Minute) // fecha e reabre uma conexão depois de 30min, mesmo se continuar em uso
```

- `SetMaxOpenConns` é o parâmetro mais importante de ajustar cedo: sem limite, um pico de tráfego pode fazer seu serviço abrir tantas conexões que o próprio banco de dados recusa novas (a maioria dos bancos tem um limite configurado de conexões simultâneas aceitas).
- `SetMaxIdleConns` evita ficar abrindo/fechando conexão toda hora em tráfego irregular, mas também não guarda um número excessivo de conexões ociosas sem uso.
- `SetConnMaxLifetime` é útil para lidar com infraestrutura que troca a máquina do banco periodicamente por trás de um balanceador de carga (bastante comum em bancos gerenciados na nuvem) — sem isso, uma conexão pode ficar "presa" apontando para uma máquina que já não existe mais.

## Executando queries: Query, QueryRow e Exec

`database/sql` tem três formas principais de rodar um comando SQL, dependendo do que você espera de volta:

```go
// QueryRow: eu espero NO MÁXIMO uma linha de volta (ex: buscar por chave primária)
row := db.QueryRowContext(ctx, "SELECT amount FROM orders WHERE id = $1", id)
var amount float64
if err := row.Scan(&amount); err != nil {
    if errors.Is(err, sql.ErrNoRows) {
        // nenhuma linha encontrada — não é bem um "erro de banco", é um resultado válido de "não achei"
    }
    return err
}

// Query: eu espero VÁRIAS linhas de volta (ex: listar registros)
rows, err := db.QueryContext(ctx, "SELECT id, amount FROM orders WHERE status = $1", "pending")
if err != nil {
    return err
}
defer rows.Close() // sempre fechar, mesmo que o loop abaixo termine mais cedo por erro

var pedidos []Order
for rows.Next() {
    var o Order
    if err := rows.Scan(&o.ID, &o.Amount); err != nil {
        return err
    }
    pedidos = append(pedidos, o)
}
if err := rows.Err(); err != nil { // erro que pode ter acontecido DURANTE a iteração, não só no início
    return err
}

// Exec: eu não espero linhas de volta, só quero rodar um comando (INSERT/UPDATE/DELETE)
result, err := db.ExecContext(ctx, "UPDATE orders SET status = $1 WHERE id = $2", "paid", id)
if err != nil {
    return err
}
linhasAfetadas, _ := result.RowsAffected() // quantas linhas o UPDATE realmente mudou
```

`Scan` preenche cada variável Go **por ponteiro**, uma célula de cada vez, na mesma ordem das colunas pedidas no `SELECT` — não existe, na stdlib, um mapeamento automático de "linha inteira vira struct inteira" (isso é o que uma biblioteca de terceiros como `sqlx` ou um ORM completo como `gorm` adiciona por cima, de forma opcional; a stdlib fica no nível mais explícito).

O `$1`, `$2` nas queries acima são **parâmetros posicionais** (a sintaxe específica do driver de Postgres; outros bancos usam `?` no lugar) — sempre use parâmetros em vez de concatenar valores direto na string SQL. Concatenar valores vindos de fora (de um usuário, de uma requisição HTTP) direto numa query SQL é a causa clássica de uma vulnerabilidade chamada **SQL injection**, onde um valor malicioso consegue alterar o comando SQL que de fato roda. Os métodos `QueryRowContext`/`QueryContext`/`ExecContext` com parâmetros posicionais protegem contra isso automaticamente, porque o valor nunca é interpretado como parte do texto do comando SQL.

## context.Context: o que é e qual problema resolve

`context.Context` é um tipo da biblioteca padrão que representa **o "tempo de vida" de uma operação em andamento** — ele carrega, ao longo de uma cadeia de chamadas de função, três coisas: um **prazo** (deadline) opcional, um sinal de **cancelamento** e, secundariamente, um jeito de carregar valores associados àquela operação (menos usado no dia a dia do que os dois primeiros).

O problema concreto que ele resolve: imagine uma requisição HTTP que chega no seu servidor, dispara uma consulta ao banco de dados, que por sua vez chama uma API externa. Se o cliente que fez a requisição HTTP original desistir (fechar a aba do navegador, cair a conexão), o ideal é que **toda a cadeia de trabalho relacionada** — a consulta ao banco em andamento, a chamada à API externa — seja cancelada também, em vez de continuar rodando até o fim para um resultado que ninguém mais vai usar. `context.Context` é o mecanismo que carrega esse sinal de "pode parar" através de todas essas camadas.

## Criando e propagando um Context

```go
ctx := context.Background() // o "contexto raiz" — usado no topo do programa (main, testes), nunca cancela sozinho

ctx, cancel := context.WithTimeout(ctx, 3*time.Second) // deriva um novo ctx com prazo de 3s
defer cancel() // sempre chame cancel, mesmo se o prazo já tiver passado — libera recursos internos do context

ctx, cancel = context.WithCancel(ctx) // deriva um ctx que só cancela quando VOCÊ chamar cancel() manualmente
defer cancel()
```

`context.Background()` é o ponto de partida — um contexto "vazio", sem prazo nem cancelamento próprio, usado no nível mais alto de um programa (dentro de `main`, ou no início de um teste). A partir dele, você **deriva** contextos filhos com `context.WithTimeout`, `context.WithDeadline` ou `context.WithCancel` — cada um desses recebe um contexto pai e devolve um novo contexto (mais uma função `cancel` que você é responsável por chamar, geralmente via `defer`, para liberar recursos internos assim que a operação terminar).

A convenção idiomática em Go é: **toda função que faz I/O (banco, rede, arquivo, chamada de API externa) recebe `ctx context.Context` como primeiro parâmetro**, e repassa esse mesmo `ctx` para qualquer chamada de I/O que ela própria faça:

```go
func (r *OrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    row := r.db.QueryRowContext(ctx, "SELECT id, amount FROM orders WHERE id = $1", id)
    var o Order
    if err := row.Scan(&o.ID, &o.Amount); err != nil {
        return nil, err
    }
    return &o, nil
}

func (s *OrderService) BuscarPedido(ctx context.Context, id string) (*Order, error) {
    return s.repo.FindByID(ctx, id) // o MESMO ctx recebido é repassado adiante, sem trocar
}
```

## Checando cancelamento e prazo esgotado

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

order, err := repo.FindByID(ctx, id)
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        // a query não terminou dentro dos 3 segundos — banco lento, rede congestionada, etc.
        log.Println("busca de pedido excedeu o prazo")
    }
    return err
}
```

Quando um `ctx` com prazo estoura, ou quando alguém chama a função `cancel` associada a ele manualmente, qualquer operação de I/O que estava usando esse `ctx` (uma query de banco em andamento, por exemplo) é interrompida, e a função que estava esperando por ela recebe um `error` — `context.DeadlineExceeded` (esgotou o prazo) ou `context.Canceled` (foi cancelado manualmente). O driver do banco de dados é quem, por dentro, escuta esse sinal e aborta a query em andamento — é por isso que passar `ctx` para `QueryContext`/`ExecContext` (em vez das versões antigas sem `Context` no nome, como `db.Query` sem contexto) importa de verdade, não é só um parâmetro decorativo.

## Cancelamento em cascata: o gotcha mais comum

Se o `ctx` da requisição HTTP original for cancelado, **todo `ctx` derivado dele** (o que foi passado para a query de banco, o que foi passado para a chamada de API externa) também cancela automaticamente, em cascata — desde que a cadeia inteira de funções realmente propague o `ctx` recebido, sem trocá-lo no meio do caminho.

```go
// ERRADO: "esquece" o ctx recebido e usa um novo, desconectado da cadeia de cancelamento
func (r *OrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    row := r.db.QueryRowContext(context.Background(), "SELECT ...", id) // ctx recebido foi ignorado!
    // ...
}

// CERTO: propaga o ctx recebido, mantendo a cadeia de cancelamento intacta
func (r *OrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    row := r.db.QueryRowContext(ctx, "SELECT ...", id)
    // ...
}
```

Trocar `ctx` por `context.Background()` no meio de uma cadeia de chamadas é um erro fácil de cometer e difícil de perceber olhando só aquela função isolada — o código compila e funciona normalmente no caminho feliz. O problema só aparece sob carga real: uma query que deveria ter sido cancelada continua rodando até o fim, consumindo tempo de banco e memória para um resultado que já não importa mais para ninguém, porque o cliente que pediu já desistiu.

## Exercício

Sem exercício gerado por padrão neste tópico — quando quiser praticar, rode `/exercise acesso a dados e context.context`; o código vai em `exercise/` (fora do git, ver `.gitignore`). Um bom primeiro exercício: escrever uma função que roda uma query lenta de propósito (ex: `SELECT pg_sleep(5)`) com um `context.WithTimeout` de 2 segundos, e confirmar que o erro devolvido é `context.DeadlineExceeded`.
