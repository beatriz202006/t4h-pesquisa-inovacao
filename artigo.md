# Otimização de desempenho de uma aplicação com PostgreSQL

## 1. Diagnóstico do problema

O desafio propõe avaliar o desempenho de uma aplicação de loja virtual composta por uma API desenvolvida em Node.js 20 com Express e um banco de dados PostgreSQL 16, executados por Docker Compose. O ambiente possui recursos limitados: 0,5 CPU e 256 MB de memória para a API, e 1 CPU e 512 MB para o banco. A base sintética contém 100 mil clientes, 20 mil produtos, 500 mil pedidos e 1,5 milhão de itens de pedido.

A avaliação inicial utilizou o script de carga fornecido, com 20 usuários virtuais, 15 segundos de aquecimento e 60 segundos de carga. Foram realizadas três execuções. A mediana dos resultados indicou vazão de 4,67 requisições por segundo e tempo de resposta mediano geral de 1.001,1 ms. A rota de consulta dos pedidos de um cliente apresentou mediana de 12.156,3 ms. Durante os testes, o uso de CPU do banco permaneceu próximo de 100%.

A análise dos planos de execução identificou varreduras sequenciais em tabelas utilizadas por consultas frequentes, como `pedidos`, `itens_pedido` e `clientes`. Também foi identificado o padrão N+1 na rota de pedidos de um cliente: consultas adicionais eram executadas para recuperar os itens de cada pedido e seus respectivos produtos.

Essas evidências direcionaram a investigação para otimizações no acesso aos dados e na quantidade de consultas realizadas, preservando o comportamento da API.

## 2. Estado da arte

A análise de desempenho de consultas relacionais pode ser apoiada pelo comando `EXPLAIN`, que apresenta o plano de execução escolhido pelo PostgreSQL e permite identificar operações como varreduras sequenciais, buscas por índice e estratégias de junção (PostgreSQL Global Development Group, s.d.-a).

Os índices são estruturas utilizadas para acelerar a recuperação de registros, mas também acrescentam custos de armazenamento e manutenção. Índices compostos podem atender consultas que utilizam mais de uma coluna em filtros e ordenações, sendo importante considerar a ordem das colunas (PostgreSQL Global Development Group, s.d.-b). Já os índices sobre expressões permitem indexar resultados de funções, como `lower(email)`, atendendo buscas que desconsideram diferenças entre letras maiúsculas e minúsculas (PostgreSQL Global Development Group, s.d.-c).

Outra estratégia é reduzir consultas repetidas para recuperar dados relacionados. Turcotte, Aldrich e Tip (2022) investigaram o problema N+1 em aplicações que utilizam ORM e apresentaram uma técnica de refatoração para substituir consultas repetidas por consultas mais abrangentes. Embora o estudo tenha como foco aplicações com ORM, o padrão de consultas repetidas também pode ocorrer em aplicações que utilizam SQL diretamente.

Para avaliar o desempenho, a documentação do k6 apresenta métricas como quantidade de requisições, taxa de falhas e duração das respostas, incluindo percentis (Grafana Labs, s.d.). Essas medidas permitem observar diferentes aspectos do comportamento da aplicação sob carga.

Essas referências fundamentaram a análise dos planos de execução, a escolha dos índices e a investigação da redução de consultas repetidas.

## 3. Comparação de alternativas

Foram aplicadas quatro estratégias de otimização, direcionadas aos problemas identificados:

**1. Índice composto em `itens_pedido`:** foi criado um índice sobre `(pedido_id, id)`, adequado à consulta que filtra os itens pelo pedido e os ordena pelo identificador. A alteração substituiu a varredura sequencial e a ordenação por uma busca utilizando índice.

**2. Índice composto em `pedidos`:** foi criado um índice sobre `(cliente_id, criado_em DESC, id DESC)`, considerando o filtro por cliente e a ordenação decrescente dos pedidos. A estrutura foi definida de acordo com os critérios utilizados pela consulta.

**3. Índice sobre `lower(email)`:** foi criado um índice de expressão para atender à consulta que utiliza `lower()` na comparação de e-mails. Um índice convencional sobre a coluna não atenderia diretamente à expressão utilizada.

**4. Reescrita da rota de pedidos:** a implementação original executava consultas adicionais para recuperar itens e produtos. A alteração substituiu essa sequência por uma consulta com `JOIN`, seguida da organização dos registros em memória para preservar a estrutura JSON da resposta.

As três primeiras estratégias modificaram o acesso aos dados por meio de índices; a quarta reduziu a quantidade de consultas executadas pela aplicação.

## 4. Justificativa da solução escolhida

As otimizações foram escolhidas com base nos planos de execução e nas medições iniciais, priorizando alterações direcionadas aos gargalos identificados.

Os índices compostos foram definidos a partir das colunas efetivamente utilizadas nos filtros e nas ordenações. Para a busca por e-mail, o índice de expressão foi escolhido porque corresponde diretamente à função utilizada na consulta.

A reescrita da rota foi priorizada devido ao custo das consultas repetidas do padrão N+1. A utilização de `JOIN` permitiu recuperar os dados relacionados em uma consulta principal, enquanto a organização em memória preservou a estrutura da resposta. Foi utilizado `LEFT JOIN` para manter também pedidos sem itens associados.

As quatro alterações eram compatíveis com as restrições do ambiente e permitiam preservar o contrato da API. Cada etapa foi validada com três execuções de carga e comparação das respostas com o contrato original.

## 5. Resultados dos experimentos

Cada cenário foi avaliado por três execuções, mantendo o mesmo ambiente e protocolo de carga. A tabela apresenta as medianas das medições:

| Cenário        | Requisições/s | Mediana geral (ms) | Pedidos do cliente (ms) |
|----------------|---------------|--------------------|-------------------------|
| Linha de base  | 4,67          | 1.001,1            | 12.156,3                |
| Experimento 01 | 7,91          | 804,8              | 4.800,0                 |
| Experimento 02 | 9,35          | 889,4              | 3.393,3                 |
| Experimento 03 | 9,69          | 405,4              | 3.196,6                 |
| Experimento 04 | 12,00         | 300,4              | 386,2                   |

No experimento 01, o índice em `itens_pedido` aumentou a vazão para 7,91 requisições por segundo. Na consulta analisada, o plano passou de `Parallel Seq Scan` para `Index Scan`, sem etapa adicional de ordenação.

No experimento 02, o índice em `pedidos` reduziu a mediana da rota de pedidos do cliente para 3.393,3 ms. A vazão chegou a 9,35 requisições por segundo, embora a mediana geral tenha aumentado em relação ao experimento anterior.

No experimento 03, o índice sobre `lower(email)` substituiu a varredura sequencial por operações `Bitmap Index Scan` e `Bitmap Heap Scan`. A mediana da rota de busca por e-mail chegou a 187,9 ms, redução aproximada de 76,9% em relação à etapa anterior.

O experimento 04 apresentou o maior ganho na rota investigada. A reescrita para eliminar o padrão N+1 reduziu sua mediana de 3.196,6 ms para 386,2 ms, aproximadamente 87,9%. A vazão chegou a 12,00 requisições por segundo.

Em relação à linha de base, o estado final apresentou aumento aproximado de 157% na vazão e redução de 70% na mediana geral. Na rota de pedidos do cliente, a redução acumulada foi de aproximadamente 96,8%.

Todas as execuções apresentaram taxa de erro de 0%. O teste de contrato confirmou que as 42 respostas verificadas permaneceram iguais às registradas originalmente.

## 6. Limitações e próximos testes

Os resultados apresentam algumas limitações. Cada cenário foi avaliado por apenas três execuções, na mesma máquina e com o mesmo perfil de carga. Além disso, as otimizações foram cumulativas, pois os índices anteriores permaneceram ativos nas etapas seguintes. Portanto, os resultados representam a evolução do conjunto de alterações, mas não isolam completamente a contribuição individual de cada índice.

Também não foram avaliados os custos dos índices em operações de escrita nem o espaço adicional ocupado por essas estruturas. Como os testes priorizaram consultas de leitura, não é possível concluir como o desempenho se comportaria em cargas com inserções, atualizações e exclusões frequentes.

O banco continuou apresentando utilização de CPU próxima de 100%, e consultas de pedidos por status e relatórios de vendas por dia permaneceram com tempos elevados. Como próximos passos, recomenda-se investigar seus planos de execução com `EXPLAIN (ANALYZE, BUFFERS)` e utilizar ferramentas como `pg_stat_statements` para identificar consultas com maior consumo de recursos.

Também seriam relevantes testes com mais execuções, diferentes níveis de concorrência, volumes maiores de dados e cargas mistas de leitura e escrita.

## 7. Referências e declaração de uso de IA

### Referências

GRAFANA LABS. *Metrics*. Grafana k6 Documentation. s.d.

POSTGRESQL GLOBAL DEVELOPMENT GROUP. *EXPLAIN*. PostgreSQL 16 Documentation. s.d.-a.

POSTGRESQL GLOBAL DEVELOPMENT GROUP. *Multicolumn Indexes*. PostgreSQL 16 Documentation. s.d.-b.

POSTGRESQL GLOBAL DEVELOPMENT GROUP. *Indexes on Expressions*. PostgreSQL 16 Documentation. s.d.-c.

POSTGRESQL GLOBAL DEVELOPMENT GROUP. *Indexes*. PostgreSQL 16 Documentation. s.d.-d.

TURCOTTE, Alexi; ALDRICH, Mark W.; TIP, Frank. *Reformulator: Automated Refactoring of the N+1 Problem in Database-Backed Applications*. In: 37th IEEE/ACM International Conference on Automated Software Engineering (ASE), 2022. DOI: 10.1145/3551349.3556911.

### Declaração de uso de inteligência artificial

Ferramentas de inteligência artificial generativa foram utilizadas como apoio na compreensão de conceitos técnicos, discussão de alternativas de otimização, revisão de código e da redação. As sugestões foram analisadas pela autora. A implementação das alterações, a execução dos experimentos, a coleta dos resultados e a validação do comportamento da aplicação foram realizadas pela autora, que permanece responsável pelas decisões técnicas e conclusões apresentadas.