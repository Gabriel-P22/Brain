# Boas práticas de backend

Referência rápida e agnóstica de linguagem/framework — práticas de engenharia de backend que valem pra qualquer app deste vault, não só Go. Exemplos concretos e convenções específicas de cada linguagem/stack ficam em `config/reference/` de cada linguagem quando fizer sentido ter exemplo de código (nem toda prática aqui precisa de um).

## Design de API

Recursos como substantivo, verbo HTTP como ação (`POST /orders`, não `POST /createOrder`). Versionar desde o início (`/v1/...` ou header) — quebrar contrato sem versionar quebra todo consumidor de uma vez. Paginação em qualquer lista que possa crescer sem limite — nunca retornar "todos os registros" por padrão. Idempotência em métodos que não são naturalmente idempotentes (ex: `POST` de criação) via chave de idempotência quando o cliente pode repetir a chamada (retry de rede).

## Tratamento de erro e status code

Formato de erro consistente em toda a API (mesmo shape de JSON pra todo erro, não um formato por endpoint). Status code correto — não devolver `200` com `{"error": true}` no corpo. Nunca vazar detalhe interno (stack trace, query SQL, path de arquivo) pro cliente; log detalhado fica no servidor, resposta ao cliente é enxuta.

## Observabilidade

Log estruturado (JSON ou key-value), não string livre — permite buscar/filtrar depois. Todo request ganha um ID de correlação (gerado na entrada ou propagado se já vier de upstream) que aparece em todo log daquela requisição — sem isso, depurar um problema em produção com concorrência é adivinhação. Métricas (contagem, latência, taxa de erro) por endpoint — não espera incidente pra descobrir que precisa. Tracing distribuído quando há mais de um serviço na cadeia de uma requisição.

## Configuração

Config vem de variável de ambiente ou arquivo de config fora do código (12-factor app) — nunca hardcoded, nunca commitado. Segredo (senha, chave de API) nunca vai pro repositório, nem em exemplo "temporário" — usar `.gitignore` e um vault/secret manager em produção. Mesma imagem/binário deve rodar em qualquer ambiente (dev/staging/prod) só trocando config externa, sem rebuild.

## Segurança básica

Validar todo input que cruza a fronteira do sistema (nunca confiar em dado vindo de fora, mesmo de "outro time interno"). Autenticação (quem é você) e autorização (o que você pode fazer) são camadas separadas — não misturar a checagem das duas num lugar só. Rate limiting em endpoint público, pelo menos algo básico, pra não virar vetor de abuso/DoS acidental. Princípio do menor privilégio em credencial de serviço (banco, fila, storage) — não usar credencial de admin pra tudo.

## Resiliência

Timeout em toda chamada de rede (banco, API externa, fila) — sem timeout, uma dependência lenta trava o serviço inteiro. Retry com backoff exponencial (+ jitter) em falha transitória, nunca retry imediato em loop. Circuit breaker quando uma dependência externa está consistentemente falhando, pra parar de bater nela e deixar ela se recuperar. Graceful shutdown — capturar sinal de término e terminar requisição em andamento antes de matar o processo, não cortar no meio.

## Estratégia de testes

Pirâmide de teste: muito teste unitário (rápido, isolado), menos teste de integração (com dependência real ou próxima de real), poucos end-to-end (caro, lento, mais frágil). Regra de domínio pura deve ser testável sem subir banco/rede — se não é, geralmente é sinal de camada mal separada (ver [CLEAN-ARCHITECTURE.md](CLEAN-ARCHITECTURE.md)).

## Documentação de contrato

API com contrato explícito (OpenAPI/Swagger ou equivalente) como fonte da verdade, não descoberto lendo o código do outro time. Contrato desatualizado é pior que contrato ausente — automatizar geração a partir do código quando possível, em vez de manter os dois manualmente em sincronia.
