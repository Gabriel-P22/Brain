# Agentic loop (ReAct) e planning

Pré-requisito: [Tool/function calling e structured output](../tool-function-calling-e-structured-output/) — lá ficou estabelecido que o modelo nunca executa nada sozinho, só pede. Este tópico é o laço de controle em volta disso que transforma "um LLM que pede uma tool" em "um agente que resolve uma tarefa em várias etapas".

## O loop básico: pensar → agir → observar → repetir

Um agente, na definição mais simples possível, é código que repete este ciclo até a tarefa terminar:

```
1. Manda o histórico da conversa + tools disponíveis pro modelo
2. Modelo responde: texto final, OU pede uma tool call
3. Se pediu tool call: código executa a tool de verdade, anexa o resultado ao histórico
4. Volta pro passo 1 com o histórico atualizado
5. Repete até o modelo responder com texto final (sem pedir mais tool)
```

```python
def run_agent(user_message: str, tools: list, tool_impls: dict, max_iters: int = 10) -> str:
    messages = [{"role": "user", "content": user_message}]

    for _ in range(max_iters):
        response = client.messages.create(
            model="claude-sonnet-5", max_tokens=1024,
            tools=tools, messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return response.content[0].text  # terminou: resposta final em texto

        tool_call = next(b for b in response.content if b.type == "tool_use")
        result = tool_impls[tool_call.name](**tool_call.input)
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result", "tool_use_id": tool_call.id, "content": str(result)}],
        })

    raise RuntimeError("agente excedeu o número máximo de iterações")
```

Não há nada além disso estruturalmente — o "agente" é esse `for` com um limite de iterações (proteção contra loop infinito, análoga a um timeout de qualquer processo de longa duração), mais o histórico crescendo a cada volta. Toda a "inteligência" de decidir qual tool chamar e quando parar vem do modelo interpretando o histórico a cada chamada — o código que orquestra o loop é deliberadamente simples.

## ReAct: Reasoning + Acting

**ReAct** é o padrão (descrito formalmente num paper de 2022, mas hoje o comportamento padrão da maioria dos modelos com tool use) de intercalar **raciocínio em texto** com **ação** (tool call) a cada passo, em vez de o modelo decidir a ação direto sem "pensar em voz alta" antes. Na prática, isso aparece como o modelo gerando um trecho de raciocínio ("preciso primeiro buscar X para depois calcular Y") antes de emitir a tool call — o mesmo mecanismo de chain-of-thought já visto em [Contexto, sampling e prompt engineering](../contexto-sampling-e-prompt-engineering/), agora intercalado com ações reais em vez de só geração de texto.

Por que isso melhora resultado: forçar o modelo a articular o raciocínio antes de agir reduz a chance de pular direto para uma tool call precipitada — o mesmo motivo de chain-of-thought funcionar para problemas de múltiplas etapas, só que agora aplicado a decisão de qual ferramenta usar, não só à resposta final.

## Planning explícito vs implícito

- **Implícito** (o loop acima, "puro ReAct") — o modelo decide o próximo passo olhando só o estado atual, sem nunca escrever um plano completo antes de começar. Funciona bem para tarefas curtas/exploratórias, mas pode ziguezaguear ou perder o fio em tarefas longas, porque não há um objetivo de médio prazo guiando cada passo.
- **Explícito** — antes de agir, o modelo (ou um passo separado) gera um plano estruturado com etapas ("1. buscar X, 2. validar Y, 3. gerar relatório com X e Y"), e o loop de execução vai marcando cada etapa como concluída, com o plano inteiro reentrando no contexto a cada iteração como referência. Isso é análogo à diferença entre navegar por decisão local greedy vs ter um roteiro definido de antemão — planning explícito custa mais tokens/uma chamada a mais no início, mas reduz o risco de a tarefa "se perder" no meio de muitas iterações. É o mesmo princípio por trás do modo de planejamento de agentes de código (ver [O que é um harness](../o-que-e-um-harness/)) — gerar e validar o plano antes de tocar em qualquer efeito colateral real.

## Limites do loop

- **Custo cresce com o número de iterações** — cada volta do loop é uma chamada completa de LLM, cobrando o histórico inteiro acumulado até ali (mitigado parcialmente por prompt caching, ver [APIs de LLM e streaming](../apis-de-llm-e-streaming/)).
- **Erros compostos** — se uma tool call intermediária falha ou retorna dado errado, esse erro entra no histórico e influencia todas as decisões seguintes; não existe "rollback" automático de raciocínio.
- **`max_iters` é uma proteção real, não cosmética** — sem limite, um agente pode entrar em ciclo (chamar a mesma tool repetidamente sem progresso) e nunca convergir para uma resposta final, consumindo tempo e orçamento indefinidamente.

Quando uma única "linha" de raciocínio-ação não é suficiente — a tarefa se beneficia de dividir em sub-tarefas paralelas ou especializadas — entra o próximo tópico, [Multi-agent e orquestração](../multi-agent-e-orquestracao/).
