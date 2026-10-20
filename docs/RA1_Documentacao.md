# PjBL – RA1: Ingestão e Preparação dos Dados
**Projeto:** O que determina o desempenho e o valor dos veículos do GTA V?
**Equipe:** _(preencher)_
**Implementação:** [notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb](../notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb) (PySpark no Google Colab)

---

## 1. Definição do problema e perguntas analíticas

### Contexto
GTA V tem um catálogo de centenas de veículos (carros, motos, aeronaves, barcos), adicionados ao longo de cerca de 10 anos de atualizações (DLCs). Cada veículo tem:
- métricas de desempenho: velocidade máxima, aceleração, frenagem, tração e agilidade;
- atributos físicos de dirigibilidade (*handling*): massa, força do motor, marchas, freio e suspensão;
- um valor monetário dentro do jogo.

### Problema
Não é claro o que determina o desempenho e o valor de um veículo. Também não se sabe se os veículos lançados em atualizações recentes são sistematicamente superiores (*power creep*). A pergunta interessa a quem escolhe em qual veículo investir no jogo e a quem estuda o balanceamento de jogos.

### Perguntas analíticas
| # | Pergunta | Variáveis principais |
|---|---|---|
| P1 | Como velocidade máxima, aceleração e frenagem variam entre **classes** e **tipos** de veículo? | `vehicle_class`, `vehicle_type`, `max_speed_kmh`, `acceleration`, `max_braking` |
| P2 | Existe relação entre **valor monetário** e desempenho? Quais classes e fabricantes oferecem a melhor relação desempenho/valor? | `monetary_value`, métricas de desempenho |
| P3 | Os atributos de *handling* (massa, força do motor, marchas, freio) explicam o desempenho? | `mass_kg`, `drive_force`, `drive_gears`, `drive_max_flat_vel`, `brake_force` |
| P4 | Quais fabricantes concentram os veículos mais rápidos e mais caros? | `manufacturer` |
| P5 | Veículos de DLCs têm desempenho e valor superiores aos do jogo base (*power creep*)? | `dlc_name` |

---

## 2. Fontes de dados

| Item | `vehicles.json` | `vehicleHandlings.json` |
|---|---|---|
| Fonte e origem | Repositório público [loopgazer/GTA.V.Dataset](https://github.com/loopgazer/GTA.V.Dataset), um fork de DurtyFree/gta-v-data-dumps. Os dados foram extraídos automaticamente dos arquivos do jogo, versão v2944 (Online 1.67). | mesmo repositório |
| Referência fixa | commit `8ff8268bac0d86a7e384f8b66d6b8dc5dd7c1eea` (21/06/2023) | idem |
| Formato | JSON (array de objetos com campos aninhados) | JSON (array de objetos com campos aninhados) |
| Registros | 815 | 747 |
| Granularidade | 1 registro = 1 modelo de veículo | 1 registro = 1 perfil de dirigibilidade |
| Principais atributos | `Name`, `DisplayName`, `Manufacturer`, `Class`, `Type`, `DlcName`, `HandlingId`, `Seats`, `MonetaryValue`, `Price`, `MaxSpeed`, `Acceleration`, `MaxBraking`, `MaxTraction`, `Agility`, `Flags`, `DimensionsMin/Max` | `Id`, `Mass`, `InitialDriveForce`, `InitialDriveMaxFlatVel`, `InitialDriveGears`, `DriveBiasFront`, `BrakeForce`, `TractionCurveMax/Min`, `SuspensionForce`, multiplicadores de dano |
| Relação com o problema | Fornece as categorias (P1, P4, P5), o valor (P2) e as métricas de desempenho (P1–P5) | Fornece as variáveis físicas que podem explicar o desempenho (P3) |

**Ligação entre as fontes:** `vehicles.HandlingId` ↔ `vehicleHandlings.Id`. A relação é N:1, porque variantes de um mesmo modelo compartilham o *handling*.

**Escopo:** os demais arquivos do repositório (sons, animações, cutscenes, objetos do mapa) são listas de referência para *modding*. Não têm atributos ligados ao problema e ficaram de fora.

---

## 3. Ingestão dos dados

O processo é 100% reproduzível por código (notebook, seção 3):

1. **Ambiente.** O notebook roda no Google Colab, que já traz o PySpark. A primeira célula instala o PySpark só se ele não estiver presente. Uma `SparkSession` local é criada.
2. **Download (camada *raw*).** Os arquivos são baixados de `raw.githubusercontent.com` usando o **hash do commit**, não o nome do branch. Assim, atualizações futuras do repositório de origem não alteram os dados do projeto.
3. **Rastreabilidade.** Para cada arquivo são registrados URL, tamanho e hash SHA-256 em `data/raw/ingestao_log.json`.
4. **Leitura com PySpark.** A leitura usa `spark.read.option("multiLine", True).option("mode", "FAILFAST").json(...)`:
   - `multiLine` é necessário porque cada arquivo é um único array JSON;
   - `FAILFAST` interrompe a leitura se houver registro malformado, em vez de gerar linhas corrompidas em silêncio;
   - a coluna técnica `_ordem_origem` registra a posição de cada registro no arquivo e é usada na deduplicação.

Resultado da leitura: `vehicles` com 815 linhas × 53 colunas e `vehicleHandlings` com 747 linhas × 48 colunas.

---

## 4. Compreensão e avaliação da qualidade dos dados

| Aspecto | Resultado da avaliação |
|---|---|
| Tipos de dados | **Sem problema.** O Spark inferiu corretamente os campos numéricos (`bigint`/`double`) e os booleanos. |
| Estrutura | Há muitos campos aninhados: nomes em 15 idiomas (struct), dimensões X/Y/Z (struct) e listas (`Flags`, `Bones`, `DefaultColors`). Também há várias colunas técnicas do motor do jogo (`Hash`, `Bones`, `LayoutId`...). |
| Valores ausentes | `Manufacturer` vem **vazio ("")** em 149 veículos (18%). São veículos genéricos, sem marca no jogo (utilitários, aeronaves, emergência, trens, bicicletas). Como o valor é string vazia e não nulo, uma contagem simples de nulos não o detecta. 4 *trailers* não têm nome em inglês. |
| Valores inválidos / sentinelas | `Price` não é um preço: 493 veículos têm `100` e 314 têm `-1` (99% do total). O valor econômico real está em `MonetaryValue`. As métricas de desempenho usam **`-1` como sentinela** em 30 veículos: 25 *trailers*, que não têm motor, e 5 carros do DLC `mpg9ec` (edição *Expanded & Enhanced*), cujas estatísticas não foram extraídas. Esses 5 carros também têm `MonetaryValue = 0`. Já `MaxTraction = 0` em aeronaves e barcos é legítimo. |
| Duplicatas | `vehicles` não tem duplicatas: `Name` é único. Os 48 nomes de exibição repetidos são variantes legítimas, como `boxville`, `boxville2` etc. Em `vehicleHandlings`, **24 `Id` aparecem duas vezes**: 19 são cópias exatas e 5 têm pequenas diferenças (redefinição por um DLC posterior). Sem tratamento, essas duplicatas multiplicariam veículos no join. |
| Formatos inconsistentes | **Caixa inconsistente nas chaves:** 94 valores de `Name` têm maiúsculas, e **78 veículos não encontram o seu *handling*** numa comparação exata (`ALPHA` × `Alpha`). Todos casam depois da padronização da caixa. |
| Categorias inconsistentes | `Manufacturer` tem **códigos truncados**: `BENEFAC`/`BENEFACTOR`, `LAMPADA`/`LAMPADATI` e `UBERMACH`/`UBERMACHT` são o mesmo fabricante. `Type`, `Class` e `DlcName` são consistentes, e `Class` ↔ `ClassId` é 1:1. |
| Outliers | Pelo critério IQR, os outliers globais vêm do **tipo de veículo**: aviões chegam a cerca de 110 m/s e têm aceleração 20× maior que a dos carros. Analisados dentro de cada `Type`, os poucos que restam são valores plausíveis do jogo, como carros de corrida ou veículos militares caros. |
| Integridade referencial | 12 *handlings* não pertencem a nenhum veículo (sobras de versões antigas). O compartilhamento de *handling* entre veículos (76 casos) é esperado. |

### Resumo dos problemas
| # | Problema | Volume | Relevância |
|---|---|---|---|
| Q1 | `Manufacturer` vazio | 149 | Alta (P4) |
| Q2 | `DisplayName` ausente | 4 | Baixa |
| Q3 | `Price` é *placeholder* | 807 | Alta (P2) |
| Q4 | Sentinela `-1` / `0` nas métricas e no valor | 30 | Alta (P1–P5) |
| Q5 | Duplicatas em `vehicleHandlings` | 24 Ids | Alta (join) |
| Q6 | Caixa inconsistente nas chaves | 94 / 78 | Alta (join) |
| Q7 | Códigos de fabricante truncados | 3 fabricantes | Média (P4) |
| Q8 | Campos aninhados e colunas técnicas | ~40 colunas | Média |
| Q9 | Outliers | explicados pelo tipo | Baixa |
| Q10 | Registros fora do escopo (*trailers*) | 25 | Média |

---

## 5. Limpeza e preparação dos dados

| Procedimento | Problema | Justificativa da decisão |
|---|---|---|
| Padronizar `handling_id` em maiúsculas nas duas fontes e `name` em minúsculas | Q6 | Chaves com caixa única garantem o join: os 78 veículos que não casavam passam a casar. |
| Deduplicar `vehicleHandlings`, mantendo a última ocorrência de cada `Id` | Q5 | Para as cópias exatas, a escolha é indiferente. Para as conflitantes, assumimos que a definição posterior no arquivo é a redefinição feita por um DLC mais recente, que é a que vale no jogo. As diferenças são pequenas. Resultado: 747 → 723. |
| Remover veículos `Type = TRAILER` | Q10, Q4 | Reboques não têm propulsão (todas as métricas são `-1`), não respondem às perguntas sobre desempenho e distorceriam as estatísticas. |
| Métricas de desempenho negativas → `null`; `MonetaryValue = 0` → `null`; nova *flag* `has_performance_stats` | Q4 | `-1` e `0` significam "dado não extraído". Imputar a média inventaria desempenho para 5 carros específicos, e excluir as linhas perderia veículos válidos do catálogo. A *flag* permite filtrá-los nas análises de desempenho. |
| Remover a coluna `Price` | Q3 | A coluna é 99% *placeholder* e induziria análises erradas. A métrica de valor é `monetary_value`. |
| `manufacturer` = nome de exibição; vazio → `"Sem fabricante"` | Q1, Q7 | O nome de exibição unifica os códigos truncados. A ausência não é desconhecimento: os veículos são genéricos, sem marca no jogo. Uma categoria explícita é mais fiel que nulo e não perde linhas em agrupamentos. |
| `display_name` = nome em inglês, com o código do modelo como substituto se ausente | Q2, Q8 | Achata o struct multilíngue mantendo um único idioma de análise. |
| Achatar dimensões em `length_m`, `width_m`, `height_m` e descartar colunas técnicas | Q8 | Simplifica o conjunto e mantém só os atributos ligados às perguntas. |
| Criar `max_speed_kmh` (`MaxSpeed` × 3,6), mantendo `max_speed_ms` | padronização | `MaxSpeed` está em m/s, a unidade interna do jogo. km/h é mais interpretável. |
| Renomear as colunas para *snake_case*, com unidade quando conhecida | padronização | Nomes consistentes e autoexplicativos. |
| **Não** remover outliers | Q9 | Os outliers são explicados pelo tipo de veículo e são plausíveis. Removê-los eliminaria os veículos de alto desempenho que interessam a P1/P2. Nos próximos RAs, as comparações serão feitas dentro de cada `vehicle_type`. |
| *Left join* `vehicles` → `vehicleHandlings` por `handling_id` | integração | O veículo é a unidade de análise. *Handlings* órfãos ficam de fora naturalmente. |

---

## 6. Conjunto de dados resultante do RA1

**`vehicles_clean`**: 790 veículos × 47 colunas, 1 linha por modelo, chave `name`. Fica salvo em `data/processed/vehicles_clean.parquet` (tipos preservados) e em `vehicles_clean.csv`.

### Evidências de qualidade (validadas automaticamente no notebook, seção 6.1)
| Regra | Status esperado |
|---|---|
| `name` é chave única, não nula, em minúsculas | OK |
| O join não duplicou linhas (790 → 790) | OK |
| Todo veículo tem *handling* associado | OK |
| `manufacturer` sem nulos ou vazios; códigos truncados unificados | OK |
| `display_name` sem nulos | OK |
| Nenhuma métrica negativa e nenhum valor monetário igual a 0 | OK |
| Nulos de desempenho só onde `has_performance_stats = false` (5 veículos) | OK |
| Sem *trailers* e sem a coluna `Price` | OK |

### Antes × depois
| Indicador | Antes | Depois |
|---|---|---|
| Linhas (`vehicles`) | 815 | 790 |
| Fabricante vazio | 149 | 0 |
| Veículos sem *handling* no join | 78 | 0 |
| *Handlings* duplicados | 24 | 0 |
| Veículos com métricas sentinela (< 0) | 30 | 0 (5 com `null` + *flag*) |
| Colunas | 53 (com ~40 aninhadas ou técnicas) | 47 (planas, exceto `flags`) |

O dicionário de dados completo está na seção 6.4 do notebook.

### Pontos de atenção para o RA2
- Comparar as métricas **dentro de cada `vehicle_type`**, porque a escala muda muito entre carros e aeronaves.
- Excluir os 5 veículos com `has_performance_stats = false` das análises de desempenho.
- Para P5, associar `dlc_name` à data de lançamento de cada DLC.

> Transformações analíticas, agregações, indicadores e visualizações não fazem parte do RA1. Ficam para os próximos RAs.
