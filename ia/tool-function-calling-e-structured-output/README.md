# Tool/function calling e structured output

## O mal-entendido mais comum: o modelo não executa nada

"Tool calling" (também chamado "function calling") sugere que o modelo roda código. Não roda. Na prática, tool calling é o modelo devolvendo um **JSON estruturado** dizendo "eu gostaria de chamar a função X com esses argumentos" — e quem efetivamente executa a função é o seu código, fora do modelo. O modelo nunca toca em rede, disco ou banco de dados; ele só prevê texto (aqui, texto no formato de uma chamada de função) do mesmo jeito que prevê qualquer outro token, como visto em [O que é um LLM](../o-que-e-um-llm/).

O fluxo completo:

```
1. Você manda o prompt + a lista de tools disponíveis (schema JSON de cada uma)
2. O modelo decide: responde em texto normal, OU devolve um bloco "tool_use" com nome + argumentos
3. Se veio "tool_use": seu código executa a função de verdade
4. Você manda o resultado de volta pro modelo como uma nova mensagem
5. O modelo usa esse resultado para formular a resposta final (ou chamar outra tool)
```

```python
tools = [{
    "name": "get_weather",
    "description": "Retorna o clima atual de uma cidade",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
}]

response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Qual o clima em Curitiba?"}],
)

if response.stop_reason == "tool_use":
    tool_call = next(b for b in response.content if b.type == "tool_use")
    result = get_weather(tool_call.input["city"])  # sua função de verdade
    # ... manda `result` de volta como role "user" com tool_result, o modelo responde em texto
```

## JSON Schema é o contrato

Cada tool é descrita por um `input_schema` em JSON Schema — o mesmo formato usado para validar payload de API REST. Isso não é coincidência: é a mesma disciplina de contrato explícito que já existe em qualquer API bem desenhada (ver [contexts/common/BACKEND-BEST-PRACTICES.md](../../contexts/common/BACKEND-BEST-PRACTICES.md)) — nome, tipo, obrigatoriedade e descrição de cada campo, para que tanto o modelo quanto quem valida no seu código saibam exatamente o que esperar. Descrição ruim de campo (`description` vaga) é a causa mais comum de tool call malformada — o modelo só tem esse texto para inferir o que preencher, não vê seu código.

## Structured output

Structured output é a mesma ideia aplicada à resposta final, não a uma chamada de ferramenta: em vez de pedir "responda em JSON" em prosa livre (que pode falhar — texto extra antes/depois, campo faltando, tipo errado), você define um schema e o provedor **garante** (via decoding restrito, não só instrução) que a saída bate com esse schema.

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Extraia: João Silva, 32 anos, engenheiro."}],
    tools=[{
        "name": "extrair_pessoa",
        "input_schema": {
            "type": "object",
            "properties": {
                "nome": {"type": "string"},
                "idade": {"type": "integer"},
                "profissao": {"type": "string"},
            },
            "required": ["nome", "idade", "profissao"],
        },
    }],
    tool_choice={"type": "tool", "name": "extrair_pessoa"},  # força usar essa tool
)
```

Esse é o mesmo padrão de "pedir JSON em texto livre" do tópico anterior ([Contexto, sampling e prompt engineering](../contexto-sampling-e-prompt-engineering/)), só que validado estruturalmente em vez de na confiança — a diferença entre parsear `json.loads()` esperando que não quebre e ter uma garantia de schema antes mesmo do parse.

## Por que isso é a base de todo agente

Um "agente" (ver [Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)) não é nada além de um loop que repete o ciclo acima: modelo decide chamar uma tool → código executa → resultado volta pro modelo → repete até o modelo decidir que a tarefa terminou. Não existe mágica adicional — todo o "comportamento agente" que faz um LLM parecer estar "usando o computador" é esse loop de tool calling, com um laço de controle em volta escrito por quem construiu o agente. Entender tool calling bem é entender 80% de como qualquer harness de coding agent (Claude Code incluso, ver [O que é um harness](../o-que-e-um-harness/)) funciona por dentro.

## Design: tool como fronteira de infra

Do ponto de vista de arquitetura em camadas já familiar (ver [contexts/common/CLEAN-ARCHITECTURE.md](../../contexts/common/CLEAN-ARCHITECTURE.md)), uma tool é uma porta de entrada controlada para efeitos colaterais — o modelo nunca deveria ter acesso direto a banco de dados, sistema de arquivos ou rede; ele só pede, e o seu código, na camada de infra, decide se executa, valida o input primeiro, aplica princípio de menor privilégio (a tool só expõe o que é estritamente necessário) e loga a chamada. É a mesma disciplina de nunca confiar cegamente em input externo — aqui o "input externo" é o output do modelo, que é, na prática, tão não-confiável quanto input de usuário. Aprofundado em [Prompt injection e guardrails](../prompt-injection-e-guardrails/).
