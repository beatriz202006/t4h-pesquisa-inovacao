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
