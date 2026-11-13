# Registro de experimentos

## Experimento 01 — Índice composto em itens_pedido

### Hipótese
A consulta dos itens de cada pedido não possuía índice em `pedido_id`, causando varreduras sequenciais em uma tabela com aproximadamente 1,5 milhão de registros.

### Alteração
Criado o índice composto `(pedido_id, id)` por meio do arquivo `banco/03-indice-itens-pedido.sql`.

### Resultados

| Métrica | Linha de base | Após índice |
|---|---:|---:|
| Requisições/s | 4,67 | 7,91 |
| Mediana geral | 1001,1 ms | 804,8 ms |
| Mediana de pedidos do cliente | 12156,3 ms | 4800,0 ms |

### Variações
- Throughput: aumento aproximado de 69,4%.
- Mediana geral: redução aproximada de 19,6%.
- Mediana de pedidos do cliente: redução aproximada de 60,5%.

### Validação
As três medições após a alteração apresentaram 0% de erros. O contrato da API permaneceu igual nas 42 verificações.

O plano de execução passou de `Parallel Seq Scan` para `Index Scan`, sem operação adicional de ordenação nesse plano.

### Conclusão
Os resultados sustentam a hipótese de que o índice composto melhorou a consulta dos itens por pedido. Entretanto, a CPU do banco permaneceu próxima de 100%, indicando outros gargalos a investigar.

### Limitações
Foram realizadas três execuções por cenário. Os resultados podem variar conforme a carga e o ambiente. O custo adicional de armazenamento e escrita do índice ainda não foi medido.

## Experimento 02 — Índice composto em pedidos por cliente

### Hipótese
A consulta de pedidos por cliente não possuía índice que atendesse simultaneamente ao filtro por `cliente_id` e à ordenação por `criado_em DESC, id DESC`. Isso provocava varredura sequencial e ordenação de registros.

### Alteração
Criado o índice composto `(cliente_id, criado_em DESC, id DESC)` por meio do arquivo `banco/04-indice-pedidos-cliente-data-id.sql`.

### Resultados

| Métrica | Linha de base | Após índice |
|---|---:|---:|
| Requisições/s | 4,67 | 9,35 |
| Mediana geral | 1001,1 ms | 889,4 ms |
| Mediana de pedidos do cliente | 12156,3 ms | 3393,3 ms |

### Variações
- Throughput: aumento aproximado de 100,1%.
- Mediana geral: redução aproximada de 11,2%.
- Mediana de pedidos do cliente: redução aproximada de 72,1%.

### Validação
As três medições após a alteração apresentaram 0% de erros. O contrato da API permaneceu igual nas 42 verificações.

O plano de execução passou a utilizar `Index Scan` com o índice criado, sem operação adicional de ordenação.

### Conclusão
Os resultados sustentam a hipótese de que o índice composto melhorou a consulta de pedidos por cliente. O throughput geral também aumentou, enquanto a CPU do banco permaneceu próxima de 100%, indicando a existência de outros gargalos.

### Limitações
Foram realizadas três execuções por cenário. Os resultados podem variar conforme a carga e o ambiente. O custo adicional de armazenamento e escrita do índice ainda não foi medido.

## Experimento 03 — Índice por expressão em clientes.email

**Hipótese:** A consulta de busca de clientes por e-mail utiliza `lower(email) = lower($1)`, mas a tabela não possuía índice sobre essa expressão. Por isso, o PostgreSQL precisava percorrer os registros da tabela para encontrar o e-mail correspondente.

**Alteração realizada:** Criação de um índice por expressão:

`CREATE INDEX idx_clientes_email_lower ON clientes (lower(email));`

Arquivo: `banco/05-indice-clientes-email-lower.sql`.

**Evidência no plano de execução:** Antes da alteração, o plano apresentava `Seq Scan on clientes`, com 99.999 linhas removidas pelo filtro. Depois, passou a utilizar `Bitmap Index Scan` sobre `idx_clientes_email_lower`, seguido de `Bitmap Heap Scan`, acessando apenas um bloco da tabela.

**Resultados das medições:** Foram realizadas três execuções com 20 usuários virtuais durante 60 segundos.

- Vazão mediana: 9,69 requisições/s.
- Mediana geral de resposta: 405,4 ms.
- Mediana da rota `cliente-por-email`: 187,9 ms.
- Erros: 0% nas três execuções.
- Contrato: as 42 respostas permaneceram iguais às respostas originais.

Na comparação com o estado anterior, no qual a mediana da rota `cliente-por-email` era de 814 ms, houve redução de aproximadamente 76,9%. Em relação à linha de base original, de 1.001,1 ms, a redução acumulada foi de aproximadamente 81,2%.

**Interpretação:** Os resultados sustentam a hipótese de que o índice por expressão melhora a busca por e-mail. A alteração é especialmente adequada porque a expressão indexada corresponde à expressão utilizada no predicado da consulta. O plano também mostra que o PostgreSQL passou a localizar os registros pelo índice, evitando a varredura completa da tabela.

**Limitações:** As medições foram realizadas em três execuções na mesma máquina. Os resultados gerais incluem os índices dos experimentos anteriores, portanto não devem ser atribuídos exclusivamente a esta alteração. Também não foram medidos isoladamente o espaço adicional ocupado pelo índice nem o impacto sobre operações de escrita. O banco continuou apresentando utilização de CPU próxima de 100%, indicando que ainda existem outros gargalos.

## Experimento 04 — Reescrita da rota para eliminar consultas N+1

### Problema identificado

A rota `GET /clientes/:id/pedidos` realizava várias consultas sequenciais ao banco de dados. Primeiro, consultava o cliente e seus pedidos. Depois, para cada pedido, buscava seus itens individualmente e, para cada item, consultava novamente a tabela de produtos.

Esse padrão caracteriza o problema de consultas N+1: em vez de buscar os dados relacionados em uma consulta conjunta, a aplicação realiza diversas consultas adicionais, aumentando o número de operações e o tempo de resposta.

### Hipótese

A substituição das consultas individuais por uma consulta SQL utilizando `LEFT JOIN` entre as tabelas `pedidos`, `itens_pedido` e `produtos` reduziria a quantidade de interações com o banco de dados e, consequentemente, o tempo de resposta da rota.

### Implementação

A rota foi reescrita para buscar os pedidos, seus itens e os nomes dos produtos em uma única consulta com `LEFT JOIN`.

No código JavaScript, as linhas retornadas pelo banco são agrupadas em um `Map`, permitindo reconstruir a estrutura original da resposta, com pedidos, itens e valores totais.

Foi mantida a verificação da existência do cliente, assim como a ordenação dos pedidos e dos itens. O uso de `LEFT JOIN` também preserva pedidos que eventualmente não possuam itens associados.

### Resultados

Foram realizadas três medições com 20 usuários virtuais durante 60 segundos, mantendo o mesmo ambiente e protocolo dos experimentos anteriores.

| Métrica | Execução 1 | Execução 2 | Execução 3 | Mediana |
|---|---:|---:|---:|---:|
| Requisições/s | 12,14 | 12,00 | 11,34 | 12,00 |
| Mediana geral (ms) | 300,4 | 298,1 | 300,4 | 300,4 |
| Pedidos do cliente (ms) | 365,0 | 393,9 | 386,2 | 386,2 |
| Erros (%) | 0 | 0 | 0 | 0 |

### Comparação com o experimento 03

Em comparação com o estado anterior à reescrita da rota, registrado no experimento 03, foram observadas as seguintes diferenças:

- A vazão aumentou de 9,69 para 12,00 requisições por segundo, uma melhora de aproximadamente 23,8%.
- A mediana geral diminuiu de 405,4 ms para 300,4 ms, uma redução de aproximadamente 25,9%.
- A mediana da rota `pedidos-do-cliente` caiu de 3.196,6 ms para 386,2 ms, uma redução de aproximadamente 87,9%.

Os resultados indicam que a reescrita reduziu significativamente o tempo de resposta da rota investigada. A comparação é especialmente relevante porque os índices criados nos experimentos anteriores foram mantidos, sendo a alteração de código a principal mudança entre os dois estados avaliados.

### Validação do contrato

Após cada medição, o script de verificação foi executado. Nas três execuções, as 42 respostas permaneceram iguais às registradas na linha de base, indicando que a alteração preservou o contrato observado da API.

### Limitações e próximos passos

Apesar da melhora expressiva na rota investigada, o banco de dados continuou utilizando aproximadamente 99% a 100% da CPU durante as medições.

Outras rotas também permaneceram com tempos elevados, especialmente `pedidos-por-status` e `vendas-por-dia`. Isso indica que ainda existem gargalos de desempenho que não foram resolvidos por esta alteração.

Os testes foram realizados em um ambiente controlado, com dados sintéticos e carga de leitura. Portanto, ainda é necessário investigar os efeitos da alteração em cenários de escrita, maior volume de dados e diferentes níveis de concorrência.

**Análise complementar do plano de execução**

A execução de `EXPLAIN (ANALYZE, BUFFERS)` confirmou o uso do índice `idx_pedidos_cliente_data_id`. O PostgreSQL estimou 5 linhas e retornou exatamente 5 linhas reais.

O índice também evitou uma etapa adicional de ordenação (`Sort`), pois sua definição utiliza as colunas `criado_em DESC` e `id DESC`, correspondentes à ordenação solicitada pela consulta.

O tempo de execução observado foi de 1,694 ms nessa execução isolada. Esse resultado serve como evidência complementar do plano escolhido, mas não substitui as medições de carga realizadas com k6.
