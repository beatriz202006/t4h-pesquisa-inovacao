# Índice de evidências — diagnóstico de desempenho

## 1. Linha de base

**Objetivo:** registrar o comportamento antes das alterações.

- **Execução inválida:** `resultados/20261108-195624-00-linha-de-base-1/resumo.txt`  
  Registra 100% de erros. Não usar essa execução como comparação normal de latência.

- **Execuções válidas:**  
  `resultados/20261108-201039-00-linha-de-base-1/resumo.txt`  
  `resultados/20261108-201453-00-linha-de-base-2/resumo.txt`  
  `resultados/20261108-201740-00-linha-de-base-3/resumo.txt`

  Evidência: `pedidos-do-cliente` apresenta medianas de 11,8–12,6 s; `pedidos-por-status`, cerca de 5,0 s; `vendas-por-dia`, 5,4–6,9 s; `top-produtos`, 13,4–21,7 s. Esses endpoints já aparecem como gargalos na linha de base.

## 2. Índice de itens do pedido

**Arquivos:** os três `resumo.txt` do grupo `01-indice-itens-pedido`:

- `resultados/20261108-210418-01-indice-itens-pedido-1/resumo.txt`
- `resultados/20261108-210713-01-indice-itens-pedido-2/resumo.txt`
- `resultados/20261108-212515-01-indice-itens-pedido-3/resumo.txt`

Evidência: a mediana global cai em relação à linha de base válida, mas `pedidos-do-cliente` ainda fica em 4,7–5,1 s e `top-produtos` em 10,6–14,2 s. Portanto, essa etapa melhora o resultado geral, mas não elimina os gargalos dessas rotas.

## 3. Índice de pedidos por cliente/data/id

**Arquivos:** os três `resumo.txt` do grupo `02-indice-pedidos-cliente-data-id`:

- `resultados/20261109-225947-02-indice-pedidos-cliente-data-id-1/resumo.txt`
- `resultados/20261109-230735-02-indice-pedidos-cliente-data-id-2/resumo.txt`
- `resultados/20261109-231957-02-indice-pedidos-cliente-data-id-3/resumo.txt`

Evidência: `pedidos-do-cliente` fica em aproximadamente 3,4 s nas duas primeiras execuções e 3,5 s na terceira — melhora em relação às etapas anteriores, mas continua lento. `pedidos-por-status` permanece em torno de 4,1–4,4 s nas execuções mostradas; `top-produtos` continua acima de 10 s.

## 4. Índice de e-mail de clientes

**Arquivos:** os três `resumo.txt` do grupo `03-indice-client-email-lower`.

Evidência: a mediana de `cliente-por-email` cai de cerca de 805–1.045 ms nos testes anteriores para aproximadamente 122–193 ms nas três execuções desse grupo. Isso sustenta uma melhora observada nessa rota.

O índice de e-mail, porém, não resolve os endpoints de pedidos e relatórios: os resultados desse grupo ainda mostram `pedidos-do-cliente` na faixa de 3,1–3,3 s, `pedidos-por-status` perto de 3,9 s, `vendas-por-dia` perto de 4,0 s e `top-produtos` muito acima de 10 s.

## 5. Rewrite N1

**Arquivos:** os três `resumo.txt` do grupo `04-rewrite-n1`.

Evidência: `pedidos-do-cliente` cai para medianas de aproximadamente 365–394 ms, comparado aos 3,1–3,3 s observados no grupo anterior. É uma melhora expressiva nessa rota.

O rewrite não elimina todos os gargalos: `pedidos-por-status` permanece em cerca de 4,0–4,9 s; `vendas-por-dia`, em 5,3–7,2 s; e `top-produtos`, em 13,5–20,6 s. Além disso, as medianas globais podem parecer boas enquanto p95/p99 continuam altos; por exemplo, no segundo teste N1, a mediana global é 298,1 ms, mas o p95 é 7.416,9 ms e o p99 é 16.820,0 ms.

## 6. Uso de CPU do banco

**Arquivos:** `recursos.txt` dos grupos `02-indice-pedidos-cliente-data-id`, `03-indice-client-email-lower` e `04-rewrite-n1`.

Evidência: os relatórios indicam uso médio de CPU do banco em torno de 98,7–100,5%. O próprio relatório define 100% como um núcleo de CPU. Isso sustenta a conclusão de que o banco ficou próximo de saturar um núcleo durante esses testes.

## 7. Integridade do contrato

**Arquivos:** `contrato.txt` das execuções de `02-indice-pedidos-cliente-data-id`, `03-indice-client-email-lower` e `04-rewrite-n1`.

Evidência: os arquivos informados registram 42 respostas iguais ao contrato. Isso sustenta que, nessas execuções, a alteração de desempenho não veio acompanhada de divergência detectada pelo teste de contrato.

A primeira execução da linha de base, que apresentou respostas divergentes, deve ser registrada separadamente; não se deve generalizar esse resultado para as execuções posteriores que passaram.

## 8. Síntese das conclusões

1. A linha de base válida apresenta lentidão importante em pedidos e relatórios.
2. O índice de itens do pedido melhora o resultado global, mas mantém rotas lentas.
3. O índice de pedidos melhora `pedidos-do-cliente`, sem resolver todas as demais rotas.
4. O índice de e-mail melhora fortemente `cliente-por-email`, mas não resolve pedidos e relatórios.
5. O rewrite N1 melhora fortemente `pedidos-do-cliente`; `pedidos-por-status`, `vendas-por-dia` e `top-produtos` continuam lentos.
6. Os relatórios de recursos mostram CPU do banco próxima de um núcleo durante os testes.
7. Os testes de contrato posteriores registram 42 respostas iguais ao contrato.

**Limite da evidência:** os arquivos mostram mudanças observadas entre as execuções. Eles não isolam, por si só, a causalidade de cada alteração; variações de ambiente e de carga também podem influenciar os resultados. A comparação mais segura é por rota, usando execuções sem erros e considerando mediana, p95 e p99.