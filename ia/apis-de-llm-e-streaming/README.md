# APIs de LLM e streaming

## Request/response básico

Chamar um LLM na prática é uma chamada HTTP para uma API REST — não tem mistério de infra além disso. O corpo típico de uma request:

```python
import anthropic

client = anthropic.Anthropic()  # lê ANTHROPIC_API_KEY do ambiente
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system="Você é um assistente conciso.",
    messages=[
        {"role": "user", "content": "Explique o que é uma goroutine em uma frase."}
    ],
)
print(response.content[0].text)
```

Pontos que já valem reter:
- **`messages` é uma lista, não um objeto de "conversa"** — o histórico inteiro (todas as trocas anteriores) precisa ser reenviado a cada chamada, porque o modelo não guarda estado entre requests (ver [O que é um LLM](../o-que-e-um-llm/)). Manter a conversa é responsabilidade de quem chama a API, não do provedor.
- **`max_tokens` limita a saída**, não a entrada — é um teto de custo/latência, não um "resumo automático".
- **`role`** distingue quem gerou cada mensagem (`user`, `assistant`, e o `system` separado) — o modelo usa isso para saber o que é instrução vs o que é resposta própria anterior.

## Streaming

Por padrão, uma chamada de LLM só devolve o texto completo depois que a geração inteira termina — para uma resposta longa, isso pode significar vários segundos de espera sem nenhum feedback. **Streaming** muda isso: o servidor manda os tokens conforme são gerados, via Server-Sent Events (SSE), e o cliente processa incrementalmente.

```python
with client.messages.stream(
    model="claude-sonnet-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Liste 5 vantagens de Go para infra."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

Por que isso importa em produção: sem streaming, um endpoint HTTP tradicional (ex: um handler em Go) fica bloqueado esperando a resposta inteira do provedor antes de mandar qualquer byte para o cliente final — a percepção de latência para o usuário é a soma completa. Com streaming, o handler repassa cada chunk assim que chega (ex: via `http.Flusher` em Go, ou uma response SSE/WebSocket), e o usuário vê texto aparecendo progressivamente, como no chat da própria Claude ou ChatGPT. É a mesma motivação de qualquer streaming de resposta HTTP — não é exclusivo de LLM, só é mais crítico aqui porque a geração completa pode levar dezenas de segundos.

## Prompt caching

Quando várias chamadas reusam o mesmo prefixo grande de contexto (ex: o mesmo system prompt extenso, ou os mesmos documentos de referência em RAG), reenviar e reprocessar esse prefixo inteiro a cada chamada é caro e lento. **Prompt caching** deixa o provedor guardar o estado processado desse prefixo por um tempo (minutos), e chamadas seguintes que repetem o mesmo prefixo pagam bem menos por ele e recebem resposta mais rápida.

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "<system prompt gigante com instruções e contexto fixo>",
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": "Pergunta específica desta chamada."}],
)
```

Analogia direta: é o mesmo princípio de cache de query preparada em banco de dados, ou de memoization em Python (`functools.lru_cache`) — reprocessar o que não mudou é desperdício, então o sistema guarda o resultado intermediário e cobra/computa só o delta. A implicação de design: estruturar prompts com o conteúdo **estável** (system prompt, documentos de referência) primeiro e o conteúdo **variável** (pergunta do usuário) por último maximiza o quanto entra em cache — reordenar isso é uma otimização de custo real, não só estilo.

## Custo e pricing por token

Provedores cobram por token de **entrada** e por token de **saída**, tipicamente com preços diferentes (saída costuma custar mais que entrada, porque é o trabalho computacional caro de gerar, não só ler). Tokens de cache (leitura de cache hit) custam uma fração do preço normal de entrada; escrever no cache pela primeira vez costuma ter uma sobretaxa pequena. Modelos maiores/mais capazes custam mais por token que modelos menores — a decisão de qual modelo usar numa integração real é sempre um trade-off explícito de custo vs qualidade vs latência, análogo a escolher tamanho de instância de banco de dados: não existe "sempre o maior", existe "o adequado à tarefa".

Isso liga direto com engenharia de custo em produção — ver [Observabilidade de LLM em produção](../observabilidade-de-llm-em-producao/) para como medir esse gasto de fato, e [Integração LLM a partir de Go e Python](../integracao-llm-a-partir-de-go-e-python/) para como isso se encaixa numa chamada real de serviço.
