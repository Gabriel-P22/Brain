# Clean Code

Referência rápida e agnóstica de linguagem dos princípios de Clean Code (Robert C. Martin) — pra citar sem reexplicar do zero. Exemplos concretos e convenções específicas de cada linguagem (ex: como comentário de doc funciona, qual é o formatador padrão, como é o teste idiomático) ficam em `config/reference/` de cada linguagem: [go/config/reference/CLEAN-CODE.md](../../go/config/reference/CLEAN-CODE.md), [python/config/reference/CLEAN-CODE.md](../../python/config/reference/CLEAN-CODE.md) (e assim por diante conforme novas linguagens entrarem no vault).

## Nomes

Nomes revelam intenção. Escopo pequeno tolera (e às vezes prefere) nome curto; escopo grande exige nome descritivo. Consistência de convenção de caixa/estilo dentro do projeto importa mais que a convenção em si.

## Funções

Pequenas, um nível de abstração por função, poucos parâmetros. Parâmetro booleano isolado costuma ser sinal de que são duas funções disfarçadas de uma.

## Comentários

Comentário só quando o código não consegue se explicar sozinho (nome ruim não se resolve com comentário, se resolve renomeando). Exceção: convenções de documentação geradas automaticamente (docstring, doc comment) seguem a regra própria da linguagem/ferramenta, não a regra geral — ver arquivo da linguagem.

## Formatação

Preferir formatação automática (linter/formatter da linguagem) a debate manual de estilo — isso tira o assunto da revisão de código.

## Tratamento de erros

Tratar erro perto de onde ele acontece ou propagar com contexto — nunca engolir silenciosamente. A forma mecânica (exceção vs. valor de retorno) muda por linguagem, mas a disciplina é a mesma.

## Limites entre camadas (Boundaries)

Isolar dependência externa (SDK de terceiro, biblioteca de infra) atrás de uma abstração própria, não espalhar a API externa pelo domínio inteiro — é o Dependency Inversion Principle aplicado (ver [SOLID.md](SOLID.md#d--dependency-inversion-principle)).

## Testes

Um conceito por teste, nome que descreve o cenário, teste rápido e independente. A forma idiomática de organizar isso (framework, estilo table-driven vs. um teste por caso) muda por linguagem — ver arquivo da linguagem.

## Code smells comuns

- Classe/tipo "deus" que acumula responsabilidades demais (ver SRP).
- Abstração criada antecipadamente "pra já deixar pronto", sem um segundo caso de uso real que justifique.
- Erro/exceção ignorado silenciosamente.
- Getter/setter mecânico sem lógica nenhuma, criado só por hábito.
