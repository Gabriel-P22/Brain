# Tools, resources e prompts em MCP

Continuação de [Arquitetura MCP (client/server/host)](../arquitetura-mcp/) — lá ficou definido o papel de host/client/server. Aqui é o que um server MCP efetivamente expõe: os três primitivos do protocolo.

## Os três primitivos

- **Tools** — funções que o modelo pode chamar, com efeito (ou não) colateral: buscar dado, escrever num sistema, executar uma ação. É exatamente o tool calling já visto em [Tool/function calling e structured output](../tool-function-calling-e-structured-output/) — MCP só padroniza *como* essas tools são descobertas e chamadas através do protocolo, em vez de hardcoded no código do app.
- **Resources** — dado que o host pode ler e anexar ao contexto, sem passar pelo ciclo de "modelo decide chamar" — é o servidor expondo "aqui está um arquivo, um registro, um trecho de documentação" para o host buscar diretamente quando relevante (ex: o usuário referencia um arquivo específico, e o host busca o conteúdo via resource, sem o modelo precisar "pedir" isso como faria com uma tool).
- **Prompts** — templates de prompt reutilizáveis que o server disponibiliza, parametrizáveis (ex: um server de code review pode expor um prompt `"revisar_pr(pr_id)"` já formatado com as instruções ideais para aquele tipo de tarefa), para o usuário ou o host invocar sem reescrever o prompt do zero a cada vez.

A distinção prática entre os três: **tool** é uma ação que o modelo decide tomar dinamicamente durante o raciocínio; **resource** é dado que o host anexa ao contexto de forma mais direta/explícita; **prompt** é um template reutilizável, não dado nem ação. A maioria dos servers MCP no mercado hoje foca quase inteiramente em tools — resources e prompts existem no protocolo, mas são menos onipresentes na prática.

## Escrevendo um server simples

Um server MCP mínimo, usando o SDK oficial em Python:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("exemplo-clima")

@mcp.tool()
def get_weather(city: str) -> str:
    """Retorna o clima atual de uma cidade."""
    # ... chamada real a uma API de clima
    return f"Ensolarado, 24°C em {city}"

@mcp.resource("config://app")
def get_config() -> str:
    """Expõe configuração da aplicação como resource."""
    return "modo: produção\nregião: sa-east-1"

if __name__ == "__main__":
    mcp.run()
```

O `@mcp.tool()` já gera o schema JSON a partir da assinatura da função e docstring — o mesmo `input_schema` explícito de [Tool/function calling e structured output](../tool-function-calling-e-structured-output/), só que derivado automaticamente pelo SDK em vez de escrito à mão. Isso reforça o ponto daquele tópico: `description` clara importa — é o que o SDK usa para gerar a descrição que o modelo vê.

## Como um harness consome isso

Quando você configura um MCP server no Claude Code (ou em Claude Desktop), o harness atua como host: ele inicia (ou conecta a) o processo do server, descobre via protocolo quais tools/resources/prompts ele expõe, e passa a oferecer essas tools ao modelo lado a lado com as tools nativas do harness (Read, Bash, etc.) — do ponto de vista do loop agente ([Agentic loop (ReAct) e planning](../agentic-loop-react-e-planning/)), não existe diferença estrutural entre uma tool nativa e uma tool vinda de um MCP server: ambas são apenas entradas na lista `tools` que o modelo recebe. A configuração de conexão (comando para rodar o server, transporte — stdio local ou HTTP remoto) vive em arquivos de configuração do host (`.claude/settings.json` no caso do Claude Code), fora do escopo deste tópico teórico.

## Transporte: local vs remoto

- **stdio** — o server roda como processo filho local, comunicação via stdin/stdout. Simples, sem rede, mas só funciona na mesma máquina do host.
- **HTTP (Streamable HTTP)** — o server roda como serviço independente, possivelmente remoto, comunicação via requests HTTP. Necessário quando o server precisa ser compartilhado entre múltiplos hosts/usuários, ou já existe como serviço hospedado.

A escolha de transporte não muda o que o server expõe (tools/resources/prompts) — só como host e server trocam mensagens, análogo à diferença entre chamar uma função local vs. uma API remota: a interface pode ser idêntica, o custo de rede e falha parcial não é.
