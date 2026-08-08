# O que é um LLM

## Modelo de linguagem, em uma frase

Um LLM (Large Language Model) é uma função estatística que, dado um pedaço de texto, prevê qual é o próximo "pedaço" mais provável — e nada além disso. Tudo que parece "raciocínio", "conversa" ou "conhecimento" emerge de aplicar essa previsão repetidamente, token a token, sobre um modelo treinado em uma quantidade gigantesca de texto. Não existe um módulo separado de "lógica" ou "memória" por trás — é a mesma operação de previsão de sequência, em loop.

Isso importa na prática: quando um LLM erra um fato ou "alucina", ele não está falhando em consultar um banco de dados interno — está gerando a sequência de tokens estatisticamente mais plausível, que às vezes não corresponde à realidade. Entender isso desde o início evita o erro mais comum de quem começa: tratar o LLM como uma base de conhecimento confiável em vez de um gerador de texto plausível.

## Tokens — a unidade real de trabalho

O modelo não vê letras nem palavras inteiras — vê **tokens**, pedaços de texto (às vezes uma palavra, às vezes um pedaço de palavra, às vezes só um caractere) definidos por um vocabulário fixo do tokenizador.

```
"integração" → ["integr", "ação"]   // pode virar 2+ tokens, não 1
"the cat sat" → ["the", " cat", " sat"]   // inglês tende a tokenizar mais "limpo" que português
```

Consequências práticas:
- **Custo e limite são medidos em tokens**, não em caracteres ou palavras — 1000 tokens em português normalmente cobrem menos texto que 1000 tokens em inglês, porque idiomas com mais acentuação/conjugação fragmentam mais.
- Contagem de string (`len(s)` em Python) não corresponde a contagem de tokens — para estimar custo/limite de verdade é preciso rodar o tokenizador do provedor, não assumir.

## Embeddings, em alto nível

Cada token é convertido internamente em um **embedding**: um vetor de números (centenas a milhares de dimensões) que representa esse token — e, por extensão, textos inteiros — em um espaço onde proximidade geométrica aproxima significado semântico. "Rei" e "rainha" ficam vetorialmente próximos; "rei" e "parafuso" ficam distantes. Essa é só a intuição de alto nível aqui; o tópico [Embeddings e vector databases](../embeddings-e-vector-databases/) aprofunda o uso prático disso (busca semântica, RAG).

## Transformers e attention, sem a matemática pesada

A arquitetura por trás da maioria dos LLMs atuais é o **Transformer**. A peça central é o mecanismo de **attention** (atenção): para prever o próximo token, o modelo não olha só o token anterior — ele pondera *todos* os tokens do contexto até ali, decidindo quanto cada um "importa" para a previsão atual.

Analogia com Python: pense em `attention` como um `dict` implícito de pesos, recalculado a cada token, mapeando "cada palavra do texto até aqui" → "o quanto ela influencia a próxima palavra". Não é um dicionário literal (são operações de multiplicação de matriz, não uma busca por chave), mas a intuição de "pesar relevância de cada peça do contexto" é o que importa entender — é isso que permite ao modelo, por exemplo, resolver que "ele" numa frase se refere a um nome citado três frases atrás.

Isso é o motivo prático de existir um **context window** (janela de contexto) limitada — cada token adicional multiplica o custo de calcular attention contra todos os tokens anteriores. Detalhe aprofundado em [Contexto, sampling e prompt engineering](../contexto-sampling-e-prompt-engineering/).

## Por que "previsão de próximo token" explica o comportamento do LLM

- **Não há memória entre chamadas** por padrão — cada request é uma previsão nova a partir do texto fornecido (o "contexto"). O que parece "o modelo lembrar da conversa" é, na real, reenviar o histórico inteiro a cada chamada.
- **"Raciocínio" (chain-of-thought) é texto intermediário**, não uma etapa de lógica separada — pedir para o modelo "pensar passo a passo" funciona porque gerar passos intermediários muda a distribuição de probabilidade dos tokens seguintes, tornando a resposta final estatisticamente mais consistente. Não é o modelo "decidindo pensar antes de responder" no sentido humano.
- **Alucinação é o modo padrão, não uma falha eventual** — o modelo sempre gera o token mais plausível; quando o treino não cobriu bem aquele fato, "mais plausível" ainda produz uma frase gramaticalmente correta e confiante, só que factualmente errada. Isso é o motivo raiz de por que grounding externo (RAG, tool calling) é necessário para tarefas que dependem de fato correto — ver [Quando RAG resolve](../quando-rag-resolve/).

## Analogia com o que você já conhece

Se em Python um sistema de recomendação clássico calcula "usuário parecido com usuário" via similaridade de vetores de features, um LLM faz algo estruturalmente parecido em escala e propósito diferentes: "dado esse vetor de contexto (texto até aqui), qual o próximo vetor (token) mais próximo/provável?" — é regressão/classificação em cima de um espaço vetorial gigante, não um sistema de regras.
