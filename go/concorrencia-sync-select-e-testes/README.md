# Concorrência — sync, select e testes

## WaitGroup — esperar N goroutines terminarem

Sem `await` pra "esperar a thread acabar" como em Python. O idiom é `sync.WaitGroup`:

```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        doWork()
    }()
}
wg.Wait()   // bloqueia até todas as 3 chamarem Done()
```

`defer wg.Done()` garante que `Done()` roda mesmo se `doWork()` der panic — `defer` é o "finally" implícito do Go, executa ao sair da função, na ordem inversa de declaração.

## Mutex — quando duas goroutines mexem no mesmo dado

Channel resolve comunicação; quando o problema é "duas goroutines escrevendo na mesma variável", usa-se `sync.Mutex` — o equivalente direto do `threading.Lock` do Python:

```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

Sem lock, `go run -race` (ou `go test -race`) detecta a condição de corrida — ferramenta que Python não tem nativa equivalente pra threading.

## select — multiplexar channels

`select` é o `switch` dos channels: espera em vários ao mesmo tempo, segue o primeiro que estiver pronto.

```go
select {
case v := <-ch1:
    fmt.Println("ch1:", v)
case v := <-ch2:
    fmt.Println("ch2:", v)
case <-time.After(2 * time.Second):
    fmt.Println("timeout")
}
```

`time.After` num `case` é o padrão idiomático de timeout — não existe `try/except TimeoutError` aqui, o timeout é só mais um channel que "dispara" depois de N segundos.

## Testes em Go

`testing` é stdlib — não precisa de `pytest` externo. Arquivo `xxx_test.go`, função `func TestAlgo(t *testing.T)`, roda com `go test ./...`.

O padrão idiomático é **table-driven**: uma tabela de casos, um único `Test...` percorre todos com `t.Run` (subteste nomeado):

```go
func TestDivide(t *testing.T) {
    cases := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {"divisão normal", 10, 2, 5, false},
        {"por zero", 10, 0, 0, true},
    }

    for _, c := range cases {
        t.Run(c.name, func(t *testing.T) {
            got, err := divide(c.a, c.b)
            if (err != nil) != c.wantErr {
                t.Fatalf("erro = %v, wantErr = %v", err, c.wantErr)
            }
            if got != c.want {
                t.Errorf("got = %v, want = %v", got, c.want)
            }
        })
    }
}
```

Equivalente direto do `@pytest.mark.parametrize` — mesma ideia (uma função, N casos), sintaxe mais verbosa porque Go não tem decorator.
