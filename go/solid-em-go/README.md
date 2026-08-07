# SOLID em Go

Definição de cada princípio: [contexts/common/SOLID.md](../../contexts/common/SOLID.md). Exemplo de código por princípio: [go/config/reference/SOLID.md](../config/reference/SOLID.md). Este README não repete isso — é a síntese de como os 5 se combinam num projeto Go real, e onde dev vindo de Python costuma errar.

## Por que SOLID pega diferente em Go

SOLID nasceu em contexto de OOP com herança (Java/C++). Go não tem herança de classe — então parte do vocabulário original (ex: Liskov Substitution fala de "subclasse") precisa de tradução: em Go, "substituição" é sobre implementações de uma mesma interface, não sobre hierarquia de classe. Isso na prática torna **S**, **I** e **D** os três que mais aparecem no dia a dia Go — a ausência de herança empurra tudo pra composição + interface pequena, que é exatamente o que ISP e DIP pedem. **O** e **L** continuam válidos mas aparecem com menos frequência explícita, porque o desenho natural de Go (funções + interfaces pequenas) já evita boa parte das violações clássicas de "editar código que já funciona".

## Como os 5 se combinam num pacote pequeno

```
order/
  order.go        // struct Order + método Validate() — SRP: só sabe validar a si mesma
  repository.go    // interface OrderRepository — ISP (1 método) + DIP (abstração perto de quem usa)
  service.go        // OrderService usa OrderRepository via injeção de construtor — OCP: novo repo = nova implementação, service não muda
```

```go
// service.go
type OrderService struct {
    repo OrderRepository   // depende da interface, não de *postgres.OrderRepo
}

func NewOrderService(repo OrderRepository) *OrderService {
    return &OrderService{repo: repo}
}
```

Isso é o padrão que se repete: **struct pequeno e focado (S)**, **dependência declarada como interface pequena que o consumidor define (I + D)**, **novo comportamento = novo tipo satisfazendo a interface, sem editar o que já existe (O)**. Repository pattern completo (Módulo 3) é essa mesma ideia com mais peça.

## Erros comuns vindo de Python

- **Recriar hierarquia de classe via embedding**: usar `struct` embedding tentando simular herança (`Dog embeds Animal` pra reuso de código, não pra modelar "é um"). Embedding é composição — se o motivo é só reaproveitar método, prefira um campo nomeado + delegação explícita, ou repensar se a abstração faz sentido.
- **Interface grande "genérica" definida antes de precisar**: hábito de `ABC`/`Protocol` do Python de desenhar o contrato completo antes de ter um segundo consumidor. Em Go, o idiom é esperar o segundo caso de uso aparecer pra então extrair a interface — interface com 1 implementação e 0 alternativa em vista é abstração prematura.
- **Injeção via framework por hábito**: Python moderno às vezes usa DI framework (`dependency-injector`, etc.). Go idiomático injeta manual, via construtor, montado em `main.go` — trazer um framework de DI pra Go geralmente é over-engineering pro tamanho típico de serviço Go.
