# 🚀 Projetos para portfólio: Engenharia de Dados

> Uma trilha progressiva de **15 projetos práticos** para construir portfólio em Engenharia de Dados — do ETL básico ao Feature Store, com guias passo a passo em cada pasta.

[![Nível](https://img.shields.io/badge/nível-iniciante%20a%20avançado-blue)](#-projetos)
[![Projetos](https://img.shields.io/badge/projetos-15-green)](#-projetos)
[![Licença](https://img.shields.io/badge/licença-MIT-lightgrey)](LICENSE)

---

## 📖 Sobre o repositório

Este repositório reúne **projetos de dados** ordenados por dificuldade e complexidade. Cada projeto inclui:

- **README** com contexto, tecnologias e o que você aprende
- **Passo a passo** detalhado para implementar do zero

Use como roteiro de estudos, base para portfólio ou referência para entrevistas em Engenharia de Dados.

### Lista de projetos baseada na lista de Luiza Vieira (vbluuiza) [https://github.com/vbluuiza](https://github.com/vbluuiza/)

### acessem o canal dela no youtube [https://www.youtube.com/@vbluuiza](https://www.youtube.com/@vbluuiza)
---


## 📂 Projetos

| # | Projeto | Descrição (exemplo) |
|---|--------|---------------------|
| **01** | [ETL Básico com Arquivos Locais](projetos-ed/001-etl-basico-com-arquivos-locais) | Processar CSV/JSON de uma pasta, transformar e salvar em outro formato |
| **02** | [Pipeline End-to-End (API → DW → Dashboard)](projetos-ed/002-pipeline-end-to-end-api-dw-dashboard) | Spotify, Medium, dados públicos |
| **03** | [Framework de Testes de Qualidade de Dados](projetos-ed/003-framework-de-testes-de-qualidade-de-dados) | Comparar origem × destino, validar regras de negócio |
| **04** | [Pipeline com Orquestração Robusta](projetos-ed/004-pipeline-com-orquestracao-robusta) | DAGs com retries, SLA, cache |
| **05** | [Migração de Dados (OLTP → OLAP)](projetos-ed/005-migracao-de-dados-oltp-olap) | Banco legado → Data Lake / DW |
| **06** | [Pipeline Analítico com Camadas (Bronze / Silver / Gold)](projetos-ed/006-pipeline-analitico-com-camadas-bronze-silver-gold) | Dados brutos → dados tratados → métricas |
| **07** | [Pipeline com Change Data Capture (CDC)](projetos-ed/007-pipeline-com-change-data-capture-cdc) | Capturar mudanças em tempo real de um banco de dados transacional |
| **08** | [Projeto "Data + Aplicação"](projetos-ed/008-projeto-data-aplicacao) | Dados alimentando um app |
| **09** | [Pipeline com Processamento Distribuído](projetos-ed/009-pipeline-com-processamento-distribuido) | Spotify + Spark / Databricks |
| **10** | [Projeto com Infraestrutura como Código](projetos-ed/010-projeto-com-infraestrutura-como-codigo) | Pipeline 100% reproduzível |
| **11** | [Data Catalog e Metadata Management](projetos-ed/011-data-catalog-e-metadata-management) | Sistema para documentar, catalogar e descobrir datasets |
| **12** | [Projeto de Streaming / Near Real-Time](projetos-ed/012-projeto-de-streaming-near-real-time) | Eventos, logs, mensagens |
| **13** | [Engine de Geração Dinâmica de SQL](projetos-ed/013-engine-de-geracao-dinamica-de-sql) | SQL criado a partir de metadados |
| **14** | [Data Lineage e Observability Completa](projetos-ed/014-data-lineage-e-observability-completa) | Rastrear impacto de mudanças, monitorar saúde de pipelines, alertas inteligentes |
| **15** | [Sistema de Feature Store](projetos-ed/015-sistema-de-feature-store) | Armazenar e servir features para modelos de ML |

---

## 🗺️ Como usar

1. **Escolha um projeto** pela ordem (do 01 ao 15) ou pelo tema que quiser estudar.
2. **Entre na pasta** do projeto (ex.: `projetos-ed/001-etl-basico-com-arquivos-locais`).
3. **Leia o README** para contexto e tecnologias.
4. **Siga o passo a passo** no arquivo `passo-a-passo.md` e implemente no seu ambiente.

A lista completa com níveis, complexidade e guia de progressão está em [lista-de-projetos.md](lista-de-projetos.md).

---

## 📁 Estrutura do repositório

```
projetos-dados/
├── README.md                 ← você está aqui
├── lista-de-projetos.md      ← lista detalhada + guia de progressão
└── projetos-ed/
    ├── 001-etl-basico-com-arquivos-locais/
    │   ├── README.md
    │   └── passo-a-passo.md
    ├── 002-pipeline-end-to-end-api-dw-dashboard/
    │   ├── README.md
    │   └── passo-a-passo.md
    └── ... (até 015)
```

---

## 🎯 Trilha sugerida

| Fase | Projetos | Foco |
|------|----------|------|
| **Fundamentos** | 01–03 | ETL, pipelines completos, qualidade de dados |
| **Produção** | 04–06 | Orquestração, migração OLTP→OLAP, Bronze/Silver/Gold |
| **Especialização** | 07–10 | CDC, Data+App, Spark, IaC |
| **Excelência** | 11–15 | Catalog, streaming, SQL dinâmico, lineage, Feature Store |

---

*Se este repositório te ajudou, considere dar uma ⭐. Bom estudo e bons projetos.*
