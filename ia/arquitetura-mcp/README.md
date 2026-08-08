# Arquitetura MCP (client/server/host)

## O problema que o MCP resolve

Antes do MCP (Model Context Protocol), cada aplicação que queria dar tool use a um LLM (ver [Tool/function calling e structured output](../tool-function-calling-e-structured-output/)) implementava sua própria integração ad-hoc com cada ferramenta: um app de chat integrando com GitHub escrevia seu próprio código para chamar a API do GitHub e expor isso como tool; outro app querendo a mesma integração reescrevia tudo de novo, do zero, do seu jeito. Com M aplicações e N ferramentas, isso é M×N integrações — cada combinação exige código próprio.

O MCP padroniza o protocolo entre "app que usa LLM" e "ferramenta/fonte de dado", transformando M×N em **M+N**: qualquer app que fala o protocolo MCP (client) consegue usar qualquer ferramenta que também fale o protocolo (server), sem integração customizada por par. É estruturalmente o mesmo problema que uma interface pequena resolve em código — em vez de cada consumidor conhecer os detalhes concretos de cada implementação, todos dependem do mesmo contrato abstrato (ver Dependency Inversion em [contexts/common/SOLID.md](../../contexts/common/SOLID.md)). MCP é essa mesma ideia, só que como um protocolo entre processos/produtos diferentes, não entre pacotes de um mesmo código.

## Papéis: host, client, server

```
┌─────────────────────────────────────────┐
│  Host (ex: Claude Code, Claude Desktop)  │
│  ┌─────────────────────────────────┐    │
│  │  Client MCP (1 por server)       │    │
│  └──────────────┬────────────────────┘    │
└─────────────────┼─────────────────────────┘
                    │ protocolo MCP (JSON-RPC)
                    ▼
          ┌───────────────────┐
          │   Server MCP        │  (ex: server de GitHub, de Postgres, de Slack)
          │  expõe: tools,       │
          │  resources, prompts  │
          └───────────────────┘
```

- **Host** — a aplicação que o usuário interage diretamente (Claude Code, Claude Desktop, um IDE, um agente customizado). É quem decide *quando* consultar o LLM e *quais* servers MCP estão disponíveis para aquela sessão.
- **Client** — o componente dentro do host que mantém a conexão 1-para-1 com um server MCP específico, fala o protocolo, e traduz entre o que o host precisa e o que o server expõe. Um host com 3 integrações MCP ativas mantém 3 clients, um por server.
- **Server** — o processo (local ou remoto) que expõe as capacidades reais: acesso a um sistema (GitHub, um banco de dados, um sistema de arquivos), através de tools, resources e prompts (aprofundado no próximo tópico, [Tools, resources e prompts em MCP](../tools-resources-e-prompts-mcp/)). O server não sabe nada sobre qual LLM está sendo usado — só implementa o protocolo.

## Comparação com o que existia antes

Tool calling ad-hoc por app (o modelo anterior): cada app define suas próprias tools, com seu próprio schema, sua própria lógica de autenticação com a fonte de dado, embutida no código do app. Funciona, mas não é reutilizável — trocar de app significa reescrever a integração inteira.

MCP: a integração com uma fonte de dado (ex: "acesso a um repositório Git") é escrita **uma vez**, como um server MCP, e qualquer host compatível com o protocolo consegue usá-la sem código adicional — só configurar a conexão. Isso é o mesmo ganho de reuso que motiva escrever uma lib compartilhada em vez de duplicar lógica em cada serviço — só que aqui o "consumidor" pode ser um produto de fornecedor diferente (Claude Code, Cursor, um agente customizado), não só outro serviço do mesmo time.

## Onde isso se encaixa no que você já usa

O próprio Claude Code (o harness rodando esta sessão) é um **host** MCP — quando você conecta um MCP server (ex: um server que dá acesso a um banco de dados ou a uma API interna), o Claude Code atua como client, e as tools desse server ficam disponíveis para uso durante a sessão, junto com as tools nativas do harness. Aprofundado em [O que é um harness](../o-que-e-um-harness/), que trata do harness como categoria mais ampla (MCP é uma das formas de estender o que um harness consegue fazer, não a única).
