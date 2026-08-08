# Chunking, hybrid search e reranking

Continuação direta de [Embeddings e vector databases](../embeddings-e-vector-databases/) — lá vimos como armazenar e comparar vetores; aqui é como preparar o documento antes de indexar (chunking) e como melhorar a qualidade da busca em cima disso (hybrid search + reranking). Os três juntos são o "motor de busca" que alimenta RAG (ver [Quando RAG resolve](../quando-rag-resolve/)).

## Chunking: por que não indexar o documento inteiro

Um embedding representa um texto inteiro como **um** vetor — quanto mais longo e heterogêneo o texto, mais "diluído" e genérico esse vetor fica, perdendo capacidade de discriminar. Indexar um PDF de 50 páginas como um único embedding produz um vetor que não representa bem nenhuma seção específica dele. **Chunking** é dividir o documento em pedaços menores antes de gerar embedding de cada um, para que a busca recupere o trecho específico relevante, não o documento inteiro.

Estratégias comuns:
- **Fixed-size** — corta a cada N tokens/caracteres, com overlap entre chunks (ex: 20%) para não cortar uma frase relevante ao meio na fronteira entre dois chunks. Simples, mas ignora estrutura do texto.
- **Semantic/structural chunking** — corta respeitando a estrutura natural do documento (parágrafo, seção, heading Markdown) — um chunk é um parágrafo, não "os primeiros 500 caracteres arbitrários". Produz chunks mais coerentes semanticamente, ao custo de tamanho de chunk variável.
- **Recursive** — tenta cortar em unidades estruturais maiores primeiro (seção → parágrafo → frase) e só desce de nível quando o pedaço ainda excede o limite de tamanho.

```python
def chunk_by_paragraph(text: str, max_chars: int = 800) -> list[str]:
    paragraphs = text.split("\n\n")
    chunks, current = [], ""
    for p in paragraphs:
        if len(current) + len(p) > max_chars and current:
            chunks.append(current.strip())
            current = ""
        current += p + "\n\n"
    if current.strip():
        chunks.append(current.strip())
    return chunks
```

Trade-off central: **chunk pequeno** → busca mais precisa (o vetor representa uma ideia específica), mas perde contexto ao redor (o modelo recebe um trecho isolado, sem o parágrafo anterior que dava sentido a ele). **Chunk grande** → mais contexto por chunk, mas busca menos precisa e mais tokens gastos por chunk recuperado. Não existe tamanho ideal universal — é ajustado empiricamente por tipo de conteúdo, com avaliação (ver [Evals e LLM-as-judge](../evals-e-llm-as-judge/)).

## Hybrid search: vetorial + keyword

Busca puramente vetorial (semântica) tem um ponto cego: ela é ruim para correspondência exata — um nome próprio, um código de erro, um identificador técnico (`ORDER-4471`, `ECONNREFUSED`) não tem "significado semântico" no sentido em que embedding captura bem; ele precisa de correspondência literal. **Hybrid search** combina os dois:

- **Busca vetorial** (embedding + similaridade de cosseno) — boa para "encontre conteúdo relacionado a essa ideia", mesmo sem palavras em comum.
- **Busca por keyword** (BM25, o algoritmo clássico de ranking por termo, usado por Elasticsearch/Postgres full-text search) — boa para correspondência exata de termo/identificador.

Os dois rankings são combinados (ex: soma ponderada de score, ou Reciprocal Rank Fusion) num resultado único. Na prática, um vector DB moderno como Qdrant ou uma extensão como pgvector combinada com `tsvector` do Postgres já suportam esse híbrido nativamente — não é preciso implementar os dois algoritmos do zero, só orquestrar a combinação.

## Reranking

Busca inicial (vetorial ou híbrida) tipicamente recupera um conjunto amplo de candidatos (ex: top 50) rápido, mas com precisão limitada — os algoritmos de índice aproximado (HNSW) e BM25 são otimizados para velocidade, não para o ranking mais preciso possível. **Reranking** é uma segunda passada: um modelo especializado (mais lento, mas muito mais preciso) reavalia só esses ~50 candidatos e reordena, e só os top 3-5 finais vão de fato para o prompt do LLM.

```python
import cohere  # ou voyageai.rerank, dependendo do provedor

candidatos = vector_search(query, top_k=50)  # busca rápida, ampla
reranked = cohere_client.rerank(
    query=query, documents=candidatos, top_n=5, model="rerank-v3.5"
)
```

Analogia com engenharia de sistemas já conhecida: é o mesmo padrão de "filtro grosso e barato primeiro, refinamento caro depois" que aparece em qualquer pipeline de busca com custo assimétrico — equivalente a um índice de banco filtrando 90% dos candidatos antes de aplicar uma lógica de negócio cara linha a linha só no que sobrou.

## O pipeline completo

```
documento → chunking → embedding de cada chunk → indexado em vector DB
                                                          │
query do usuário → embedding da query → hybrid search (vetorial + keyword) → top 50
                                                          │
                                                    reranking → top 5
                                                          │
                                            top 5 vira contexto no prompt do LLM
```

Esse pipeline inteiro é o que o próximo tópico, [Quando RAG resolve (e quando não resolve)](../quando-rag-resolve/), avalia criticamente — cada etapa aqui tem custo de latência e engenharia, e nem toda aplicação precisa do pipeline completo.
