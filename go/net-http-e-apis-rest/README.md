# net/http e APIs REST

## Servidor mínimo

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /orders/{id}", getOrder)
mux.HandleFunc("POST /orders", createOrder)
http.ListenAndServe(":8080", mux)

func getOrder(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"id": id})
}
```

Desde o Go 1.22 (estamos no 1.26 aqui), o `http.ServeMux` da stdlib ganhou roteamento por método + wildcard de path (`"GET /orders/{id}"`) — antes disso, tutorial nenhum de Go conseguia montar REST decente sem um router de terceiro (`gorilla/mux`, `chi`). Hoje, pra API pequena/média, a stdlib sozinha já é viável — [Gin](../gin-framework-web/) entra quando você quer o ecossistema de middleware/binding pronto, não porque a stdlib "não sabe rotear".

## Comparando com Flask/FastAPI

Onde Python usa decorator (`@app.get("/orders/{id}")`), Go registra explicitamente no mux — mais verboso, mas sem mágica de reflection por trás. `ResponseWriter`/`*Request` são passados explicitamente pra função, não injetados por framework.

## Resposta JSON e status code

```go
w.WriteHeader(http.StatusCreated)   // precisa vir ANTES do primeiro Write/Encode — depois disso o header já foi enviado
json.NewEncoder(w).Encode(order)
```

Esquecer que `WriteHeader` só funciona antes do corpo começar a ser escrito é um erro comum de quem vem de framework que abstrai isso (Flask/FastAPI resolvem a ordem pra você).

## Design de recurso

Ver [contexts/common/BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md#design-de-api) pra convenção de nome de rota, versionamento, paginação, idempotência — vale desde o primeiro endpoint, não só quando a API cresce.
