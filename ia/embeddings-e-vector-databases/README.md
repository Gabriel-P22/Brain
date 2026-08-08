# Embeddings e vector databases

Pré-requisito conceitual: [O que é um LLM](../o-que-e-um-llm/) já introduziu embedding como "vetor que representa significado". Este tópico é o aprofundamento prático — como gerar e armazenar esses vetores para busca de verdade, que é a base do próximo tópico ([Chunking, hybrid search e reranking](../chunking-hybrid-search-e-reranking/)) e de RAG como um todo.

## Embedding como representação vetorial de significado

Um **embedding** é a saída de um modelo especializado (diferente do LLM de geração de texto) que converte um texto inteiro em um vetor de tamanho fixo (ex: 1536 ou 3072 números). Textos com significado parecido produzem vetores próximos nesse espaço; textos sem relação produzem vetores distantes — mesmo que compartilhem poucas palavras literais.

```python
from anthropic import Anthropic
# (exemplo ilustrativo — Anthropic hoje não expõe embeddings próprios;
# Voyage AI é o provedor recomendado para uso com Claude, mesma ideia de API)

import voyageai
client = voyageai.Client()
result = client.embed(["Go usa goroutines para concorrência"], model="voyage-3")
vector = result.embeddings[0]  # ex: lista de 1024 floats
```

Isso é diferente de busca por palavra-chave tradicional: buscar "concorrência em Go" por keyword só acha documentos que contêm literalmente essas palavras; buscar por embedding acha "goroutines e channels permitem paralelismo em Go" mesmo sem nenhuma palavra em comum, porque o *significado* é próximo.

## Similaridade de cosseno

A forma padrão de medir "quão perto" dois embeddings estão é a **similaridade de cosseno** — o cosseno do ângulo entre os dois vetores, variando de -1 (opostos) a 1 (idênticos em direção). Na prática, para embeddings normalizados, valores próximos de 1 indicam alta similaridade semântica.

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

Analogia com Python: é conceitualmente o mesmo tipo de operação que calcular distância entre dois pontos para um `KNN` do scikit-learn — só que em vez de features tabulares (idade, preço, etc.), as "features" são as centenas de dimensões que o modelo de embedding aprendeu a extrair do texto.

## O que é um vector database

Comparar um vetor de consulta contra **todos** os vetores armazenados, um a um (busca exaustiva/"brute force"), funciona para milhares de documentos mas não escala para milhões — o custo cresce linearmente com o volume. Um **vector database** resolve isso com índices especializados (ex: HNSW — Hierarchical Navigable Small World) que encontram os vizinhos mais próximos aproximadamente, em tempo sublinear, trocando uma fração pequena de precisão por ganho grande de velocidade.

Opções comuns no mercado:
- **pgvector** — extensão do PostgreSQL que adiciona tipo `vector` e índices de similaridade direto no banco relacional já existente. Vantagem prática: não precisa de um sistema separado se o projeto já usa Postgres — mesma base de dados guarda dado relacional e vetor.
- **Pinecone, Qdrant, Weaviate** — bancos dedicados a vetor, otimizados para esse caso de uso especificamente, com features como filtro por metadado combinado à busca vetorial, sharding, etc. Fazem sentido quando o volume ou a exigência de latência ultrapassa o que uma extensão em cima de um banco relacional entrega confortavelmente.

```sql
-- pgvector: exemplo de índice e busca
CREATE TABLE documentos (id serial, conteudo text, embedding vector(1024));
CREATE INDEX ON documentos USING hnsw (embedding vector_cosine_ops);

SELECT conteudo, 1 - (embedding <=> $1) AS similaridade
FROM documentos
ORDER BY embedding <=> $1  -- operador de distância de cosseno do pgvector
LIMIT 5;
```

## Como isso difere de busca por índice tradicional

Um índice de banco relacional (`B-tree`, já visto na prática de acesso a dados em [go/acesso-a-dados-e-context](../../go/acesso-a-dados-e-context/)) acelera busca por **igualdade ou ordenação exata** (`WHERE id = 5`, `ORDER BY created_at`). Um índice vetorial acelera busca por **proximidade aproximada em um espaço de alta dimensão** — não existe "igual", só "mais parecido". São ferramentas complementares, não substitutas: um sistema real de RAG tipicamente filtra por metadado exato via índice tradicional (ex: `WHERE tenant_id = ?`) **e** ordena por similaridade vetorial dentro desse subconjunto, combinando os dois.

Esse cruzamento — filtro exato + ranking semântico — é exatamente o que o próximo tópico aprofunda como **hybrid search**.
