# Clean Architecture / separação em camadas

Definição da regra de dependência e das camadas: [contexts/common/CLEAN-ARCHITECTURE.md](../../contexts/common/CLEAN-ARCHITECTURE.md). Exemplo de estrutura completo: [go/config/reference/CLEAN-ARCHITECTURE.md](../config/reference/CLEAN-ARCHITECTURE.md). Este README não repete isso — é sobre por que o mecanismo de Go faz essa regra "pegar" de verdade.

## O que faz a regra de dependência pegar em Go

Em Python, nada no interpretador impede um `models.py` de importar `requests` direto — a separação de camada é 100% disciplina/convenção, fácil de violar sem aviso. Em Go, o mecanismo que dá "dente" pra regra é o mesmo de sempre: interface declarada no domínio, satisfeita implicitamente pela infra (ver Dependency Inversion em [SOLID em Go](../solid-em-go/)). Não impede fisicamente um import errado, mas o padrão idiomático (interface perto de quem consome) torna o desenho errado mais estranho de escrever do que o certo — o caminho de menor resistência já é o correto.

## O teste rápido, aplicado

"Dá pra testar o domínio sem banco/servidor no ar?" — em Go isso vira literalmente: o arquivo `domain.go` não tem `import "database/sql"` nem `import "net/http"` em lugar nenhum. É `grep import` rápido de verificar, não precisa nem rodar teste pra saber se a regra foi violada.

## Por que isso importa mais aqui do que pareceria vindo de Python

Retrofit de interface em Python é barato (linguagem dinâmica, dá pra remendar depois sem reescrever assinatura em vários lugares). Em Go, quando o projeto cresce sem essa separação, desembaraçar depois custa mais — não tem "monkey patch" de emergência. Isso não significa over-engineerizar um exercício de 20 linhas com 3 camadas — significa que, num serviço real (que é o caso da vaga: API + infra), vale desenhar a fronteira desde o início, não depois que já doeu.

## Onde isso aparece no resto do plano

[Repository pattern na prática](../repository-pattern-na-pratica/) (Módulo 3) é a camada de infra implementando a interface do domínio, com exemplo mais completo. [net/http e APIs REST](../net-http-e-apis-rest/) e [Middleware e chain de handlers](../middleware-e-chain-de-handlers/) são a camada de entrega — HTTP é só um jeito de acionar o caso de uso, poderia ser CLI ou gRPC sem mudar o domínio.
