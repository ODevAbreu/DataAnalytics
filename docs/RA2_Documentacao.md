# PjBL – RA2: Data Stream
**Projeto:** O que determina o desempenho e o valor dos veículos do GTA V?
**Implementação:** [notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb](../notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb), seções 7 a 12 (Spark Structured Streaming)

## Cenário e objetivo
O projeto usa uma base estática, então segue o **Cenário A** (chegada de registros simulada). O objetivo é monitorar a **demanda** e o **valor movimentado** em uma loja de veículos do jogo: a cada 10 minutos, quantas compras ocorreram e qual a soma do `monetary_value` dos veículos comprados. Liga-se à pergunta P2 (valor × veículos) do RA1.

## Fonte e processamento
- Os registros são **compras simuladas**: `event_id`, `event_time`, `vehicle_name`, `vehicle_class`, `vehicle_type`, `valor`.
- Veículos e valores são sorteados do `vehicles_clean` do RA1 (597 carros com valor e desempenho conhecidos). Só os horários e a escolha das compras são artificiais (semente 42).
- Os 32 registros chegam em **5 lotes**, cada um um arquivo JSON gravado numa pasta enquanto a query roda. `readStream` com schema explícito e `maxFilesPerTrigger = 1` (1 lote por micro-batch). Saída em *memory sink*, consultada após cada lote.

## Janela temporal
| Item | Valor |
|---|---|
| Tipo | *Tumbling* (sem deslocamento) |
| Duração | 10 minutos, sobre o tempo do evento (`event_time`) |
| Agregações | `count`, `sum(valor)`, `avg(valor)`, `max(valor)` |
| Modo de saída | `complete` (sem *watermark*, de propósito, para que registros atrasados atualizem janelas antigas) |

## Resultados
| Janela | Após lote 1 | Após lote 3 | Final (lote 5) |
|---|---|---|---|
| 20:00 | 8 compras / 650.000 | 8 / 650.000 | **10 / 835.000** (2 atrasados no lote 4) |
| 20:10 | – | 6 / 405.000 | 6 / 405.000 |
| 20:20 | – | **7 / 1.050.000** (pico de alto valor) | 7 / 1.050.000 |
| 20:30 | – | – | 4 / 215.000 |
| 20:40 | – | – | 5 / 267.000 |

- Cada micro-batch leu exatamente os registros do lote (8, 6, 7, 6, 5); total processado = total gerado (32), valor total 2.772.000.
- O pico de alto valor aparece na janela 20:20 pelo **valor total**, e não pela contagem (7 compras contra 6 na janela anterior).
- Os 2 registros atrasados do lote 4 **reescreveram a janela 20:00** (8 → 10 compras; 650.000 → 835.000).

## Limitações
Os eventos são simulados e `monetary_value` é um valor por faixas do catálogo (25 mil, 35 mil, 50 mil, 150 mil...), não um preço de loja. Uma única janela foi usada e não há *watermark*; em produção seria necessário para limitar o estado guardado.
