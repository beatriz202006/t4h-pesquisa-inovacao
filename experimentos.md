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

**Análise complementar do plano de execução**

A execução de `EXPLAIN (ANALYZE, BUFFERS)` confirmou o uso do índice `idx_pedidos_cliente_data_id`. O PostgreSQL estimou 5 linhas e retornou exatamente 5 linhas reais.

O índice também evitou uma etapa adicional de ordenação (`Sort`), pois sua definição utiliza as colunas `criado_em DESC` e `id DESC`, correspondentes à ordenação solicitada pela consulta.

O tempo de execução observado foi de 1,694 ms nessa execução isolada. Esse resultado serve como evidência complementar do plano escolhido, mas não substitui as medições de carga realizadas com k6.
