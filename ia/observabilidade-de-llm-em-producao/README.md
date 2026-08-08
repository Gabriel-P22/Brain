# Observabilidade de LLM em produção

Fecha o Módulo 8, junto com [Evals e LLM-as-judge](../evals-e-llm-as-judge/) (medir qualidade antes de deploy) e [Prompt injection e guardrails](../prompt-injection-e-guardrails/) (mitigar risco de segurança) — observabilidade é medir o que está acontecendo depois do deploy, em produção, com tráfego real.

## O que monitorar diferente de uma API tradicional

Uma API tradicional (ver [contexts/common/BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md)) já tem observabilidade estabelecida: latência, taxa de erro HTTP, throughput. Um sistema com LLM no meio precisa de tudo isso **mais** um conjunto de métricas específicas que não existem numa API CRUD comum:

- **Tokens por chamada** (entrada, saída, cache hit/miss) — a unidade de custo real, não bytes nem requests. Sem isso, não dá para responder "quanto está custando esse endpoint" de forma precisa.
- **Latência por chamada de LLM**, separada da latência do resto do request — uma chamada de LLM pode levar segundos, ordens de magnitude mais que uma query de banco; misturar essa latência com o resto do endpoint na mesma métrica esconde onde o tempo realmente vai.
- **Taxa de erro de tool call** — quantas vezes o modelo pediu uma tool com argumento inválido, ou uma tool falhou na execução. Isso não existe em API tradicional porque não há "o modelo decidindo errado o que chamar" — é uma classe de erro nova, específica de sistemas agentic.
- **Taxa de retry/rate-limit hit no provedor** — sinaliza se o volume de chamadas está batendo em limite do provedor, distinto de erro de aplicação.
- **Distribuição de `stop_reason`** (respondeu normal, atingiu `max_tokens`, parou por tool use) — atingir `max_tokens` com frequência é sinal de que a resposta está sendo cortada no meio, um bug silencioso que não aparece como erro HTTP.

## Tracing de chamada de LLM/agente

Para um agente com múltiplas iterações (ver [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)), uma única "operação" do ponto de vista do usuário vira várias chamadas de LLM encadeadas, cada uma com suas próprias tool calls. **Tracing** captura essa árvore inteira — não só "a operação levou 8 segundos", mas "chamada 1 (2s) → tool X (1s) → chamada 2 (3s) → tool Y (0.5s) → chamada 3 (1.5s)" — permitindo identificar exatamente qual etapa é o gargalo ou onde um erro se originou, em vez de só saber que a operação como um todo falhou ou foi lenta.

```python
import logging

logger = logging.getLogger("llm_trace")

def traced_complete(prompt: str, trace_id: str, step: str) -> str:
    start = time.time()
    response = client.messages.create(model="claude-sonnet-5", max_tokens=1024,
                                        messages=[{"role": "user", "content": prompt}])
    logger.info("llm_call", extra={
        "trace_id": trace_id, "step": step,
        "latency_ms": (time.time() - start) * 1000,
        "input_tokens": response.usage.input_tokens,
        "output_tokens": response.usage.output_tokens,
        "stop_reason": response.stop_reason,
    })
    return response.content[0].text
```

O `trace_id` correlacionando todas as chamadas de uma mesma operação é o mesmo princípio de correlation ID em qualquer sistema distribuído — só que aqui a "cadeia distribuída" é o loop de raciocínio de um agente, não microserviços.

## Logging de prompt/resposta/custo

Guardar prompt e resposta de cada chamada (com atenção a dado sensível — ver considerações de PII/dado do cliente em [contexts/common/BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md)) é o que permite, depois de um incidente ou reclamação de qualidade, reconstruir exatamente o que o modelo viu e respondeu — sem isso, debugar "por que o agente fez X" é impossível retroativamente. Isso também é a fonte de dado para melhorar o dataset de eval ao longo do tempo (ver [Evals e LLM-as-judge](../evals-e-llm-as-judge/)): casos reais de produção que falharam viram novos casos de teste.

Custo acumulado (soma de tokens × preço por chamada, agregado por endpoint/usuário/dia) é o que permite responder, sem estimativa, "quanto esse feature está custando" — informação que não existe em sistema sem LLM, onde o custo marginal por request é próximo de zero e não precisa ser rastreado com esse nível de granularidade.

## Por que isso fecha o módulo

Evals medem qualidade **antes** do deploy, contra um dataset fixo. Observabilidade mede comportamento real **depois** do deploy, contra tráfego que você não controla — os dois são complementares, não substitutos um do outro: um sistema pode passar em todos os evals e ainda assim degradar em produção (o provedor atualiza o modelo, o padrão de uso real diverge do dataset de teste, um caso de injection não coberto aparece). É a mesma relação entre teste automatizado e monitoramento em produção que já existe em qualquer sistema — LLM só adiciona uma dimensão nova de métrica (token, custo, comportamento de tool call) ao que já era familiar.
