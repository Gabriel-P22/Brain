# Evals e LLM-as-judge

## Por que "parece bom" não é uma métrica

Saída de LLM é texto em linguagem natural — não existe um `assert response == expected` direto como num teste unitário tradicional (ver [go/testes-unitarios-e-integracao](../../go/testes-unitarios-e-integracao/)), porque a mesma pergunta pode ter múltiplas respostas corretas, formuladas de formas diferentes. Isso cria uma armadilha real: sem medição sistemática, a única forma de saber se um prompt/pipeline melhorou é "parece melhor quando eu testei manualmente" — que não escala, não é reprodutível, e é enviesado pelos poucos exemplos que a pessoa testou. **Evals** são o equivalente de suíte de testes para sistemas baseados em LLM: um jeito de substituir essa impressão subjetiva por número medido, repetível, comparável entre versões de prompt/modelo/pipeline.

## Anatomia de um eval

```
dataset de eval (entradas + saída esperada ou critério de qualidade)
        │
        ▼
rodar o sistema real (prompt, RAG, agente) contra cada entrada
        │
        ▼
métrica: comparar saída obtida vs esperada/critério
        │
        ▼
score agregado (% de acerto, nota média, etc.) — comparável entre execuções
```

- **Dataset** — um conjunto de casos representativos, idealmente cobrindo tanto o caminho feliz quanto casos de borda conhecidos (perguntas ambíguas, entradas malformadas, tentativas de prompt injection). Mesma disciplina de um bom conjunto de casos de teste, com o cuidado extra de incluir exemplos difíceis/adversariais, porque LLM tende a "passar" fácil em casos triviais.
- **Métrica** — depende da tarefa: para extração estruturada, comparação exata de campo é viável (é código determinístico avaliando saída determinística, mesmo com LLM no meio); para geração de texto livre (resumo, resposta aberta), comparação exata não funciona — precisa de outra abordagem, geralmente LLM-as-judge.

## LLM como avaliador de outro LLM

Para tarefas onde a saída é texto livre e não há uma única resposta "certa" bit-a-bit, uma técnica comum é usar um segundo LLM (às vezes o mesmo modelo, às vezes um mais capaz) como **juiz**: ele recebe a entrada, a saída gerada, e um critério explícito de avaliação, e devolve um julgamento estruturado (nota, pass/fail, categoria de erro).

```python
judge_prompt = """
Avalie a resposta abaixo segundo o critério: "explica o conceito corretamente,
sem inventar fato não presente na fonte fornecida".

Pergunta: {question}
Fonte: {source}
Resposta a avaliar: {response}

Responda apenas: PASS ou FAIL, seguido de uma frase justificando.
"""
```

O ponto central de por que isso funciona (e onde falha): um LLM julgando texto contra um critério explícito e bem definido é mais consistente e escalável que revisão manual, mas herda as mesmas limitações de qualquer LLM — pode ser enganado por resposta bem escrita mas factualmente errada, tende a favorecer respostas mais longas/detalhadas mesmo quando não são melhores, e sua nota não é uma verdade objetiva, é a opinião de outro modelo. Por isso, LLM-as-judge funciona melhor com um critério **estreito e explícito** (não "essa resposta é boa?", mas "essa resposta contém o fato X? sim/não") — quanto mais próximo de uma pergunta binária/verificável, mais confiável o julgamento.

## Por que isso substitui "parece bom" por medição

Sem eval, mudar um prompt (ou trocar de modelo, ou ajustar a estratégia de chunking do RAG) é uma aposta às cegas — você não sabe se a mudança melhorou ou piorou fora dos poucos casos que testou manualmente, e regressões silenciosas em casos não testados passam despercebidas. Com um dataset de eval rodando automaticamente (idealmente em CI, análogo a rodar `go test ./...` a cada mudança), toda alteração de prompt/modelo/pipeline produz um número comparável contra a versão anterior — a mesma disciplina de regressão que já existe para código tradicional, aplicada ao componente não-determinístico do sistema.

Isso conecta diretamente com o próximo tópico, [Prompt injection e guardrails](../prompt-injection-e-guardrails/) — um dataset de eval robusto inclui casos adversariais, tornando eval também uma ferramenta de detecção de regressão de segurança, não só de qualidade.
