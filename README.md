# PjBL – Data Analytics: Catálogo de Veículos do GTA V

Projeto da disciplina de Data Analytics (PjBL). O objetivo é investigar **o que determina o desempenho e o valor dos veículos do GTA V** usando os dados extraídos do jogo.

## RA1 – Ingestão e Preparação dos Dados

| Entregável | Arquivo |
|---|---|
| Notebook (PySpark, Google Colab) | [notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb](notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb) |
| Documentação (template da disciplina) | [docs/RA1_Data_Analytics.docx](docs/RA1_Data_Analytics.docx) |
| Documentação (versão Markdown) | [docs/RA1_Documentacao.md](docs/RA1_Documentacao.md) |

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ODevAbreu/DataAnalytics/blob/main/notebooks/RA1_Ingestao_Preparacao_GTAV.ipynb)

### Como reproduzir
1. Abra o notebook no Google Colab (botão acima ou `Arquivo → Abrir notebook → GitHub`).
2. `Ambiente de execução → Executar tudo`.
3. O notebook baixa os dados brutos de um **commit fixo** do repositório de origem, avalia a qualidade, limpa e grava o resultado em `data/processed/` (Parquet e CSV).

Nenhum arquivo precisa ser enviado manualmente; o PySpark já vem instalado no Colab.

### Fonte de dados
[loopgazer/GTA.V.Dataset](https://github.com/loopgazer/GTA.V.Dataset), commit `8ff8268bac0d86a7e384f8b66d6b8dc5dd7c1eea` (GTA V v2944 / Online 1.67):
- `vehicles.json`: 815 veículos (catálogo, desempenho, valor)
- `vehicleHandlings.json`: 747 perfis de dirigibilidade (massa, motor, freio, suspensão)

### Estrutura
```
├── notebooks/   notebook do RA1 (Colab + PySpark)
├── docs/        documentação do processo analítico
└── data/        gerado pelo notebook (raw/ e processed/); não versionado
```

### Fluxo
Fontes de Dados → Ingestão → Compreensão dos Dados → Avaliação da Qualidade → Limpeza e Preparação → **`vehicles_clean`** (790 veículos × 47 colunas) → RA2
