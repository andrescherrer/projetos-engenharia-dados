# Lista de Projetos de Engenharia de Dados + O que Você Aprende

> **Nota do Especialista:** Esta lista está organizada por ordem crescente de dificuldade e complexidade, criando uma jornada de aprendizado progressiva. Cada projeto constrói sobre os conceitos aprendidos nos anteriores.

---

## 1. ETL Básico com Arquivos Locais
**Nível:** Iniciante | **Complexidade:** ⭐

**Exemplo:** Processar CSV/JSON de uma pasta, transformar e salvar em outro formato

**Você trabalha com:**
- Python (pandas/polars)
- Arquivos locais (CSV, JSON, Parquet)
- Estruturas de dados básicas
- Scripts Python simples

**Você aprende:**
- Conceitos fundamentais de ETL (Extract, Transform, Load)
- Manipulação básica de dados
- Estrutura de um pipeline simples
- Debugging e tratamento de erros básicos

---

## 2. Pipeline End-to-End (API → DW → Dashboard)
**Nível:** Iniciante/Intermediário | **Complexidade:** ⭐⭐

**Exemplo:** Spotify, Medium, dados públicos

**Você trabalha com:**
- APIs REST / GraphQL
- Python (requests, pandas/polars)
- SQL
- Data Warehouse (Postgres, BigQuery, Redshift)
- Orquestração (Airflow / Prefect / Dagster)
- Visualização (Grafana / Power BI / Superset)

**Você aprende:**
- Fluxo completo de dados (ingestão → transformação → consumo)
- Estrutura de pipelines reais
- Versionamento e organização de código
- Pensar em dados para consumo analítico

---

## 3. Framework de Testes de Qualidade de Dados
**Nível:** Iniciante/Intermediário | **Complexidade:** ⭐⭐

**Exemplo:** Comparar origem × destino, validar regras de negócio

**Você trabalha com:**
- SQL
- Python
- Testes automatizados
- Data Quality (checks)

**Você aprende:**
- Confiabilidade de dados
- Detecção de inconsistências
- Validação pós-pipeline
- Mentalidade data-driven
- **Por que é importante:** Aprender a validar dados desde cedo evita problemas graves depois

---

## 4. Pipeline com Orquestração Robusta
**Nível:** Intermediário | **Complexidade:** ⭐⭐⭐

**Exemplo:** DAGs com retries, SLA, cache

**Você trabalha com:**
- Airflow / Prefect
- DAGs, tasks, sensors
- Logs, retries, schedules
- Ambientes (dev/prod)

**Você aprende:**
- Orquestração como produto
- Tratamento de falhas
- Observabilidade de pipelines
- Boas práticas de produção

---

## 5. Migração de Dados (OLTP → OLAP)
**Nível:** Intermediário | **Complexidade:** ⭐⭐⭐

**Exemplo:** Banco legado → Data Lake / DW

**Você trabalha com:**
- Bancos relacionais
- Estratégias de extração (full, incremental, CDC)
- Modelagem analítica
- Validação de dados

**Você aprende:**
- Arquitetura corporativa
- Trade-offs de migração
- Garantia de consistência
- Governança de dados
- Diferenças entre sistemas transacionais e analíticos

---

## 6. Pipeline Analítico com Camadas (Bronze / Silver / Gold)
**Nível:** Intermediário | **Complexidade:** ⭐⭐⭐

**Exemplo:** Dados brutos → dados tratados → métricas

**Você trabalha com:**
- Delta Lake
- SQL analítico
- Modelagem dimensional
- Incremental load

**Você aprende:**
- Arquitetura Lakehouse
- Organização de dados para negócio
- Performance analítica
- Governança e escalabilidade
- Medallion Architecture (padrão da indústria)

---

## 7. Pipeline com Change Data Capture (CDC)
**Nível:** Intermediário/Avançado | **Complexidade:** ⭐⭐⭐
**(A IA sugeriu isso)**

**Exemplo:** Capturar mudanças em tempo real de um banco de dados transacional

**Você trabalha com:**
- Debezium / AWS DMS / Fivetran
- Kafka Connect
- Logs de transação (WAL)
- Sincronização incremental

**Você aprende:**
- Captura de mudanças em tempo real
- Sincronização incremental eficiente
- Minimizar impacto em sistemas fonte
- Padrões de replicação de dados
- **Por que é importante:** CDC é essencial para sistemas que precisam de dados atualizados sem fazer full load

---

## 8. Projeto "Data + Aplicação"
**Nível:** Intermediário | **Complexidade:** ⭐⭐⭐

**Exemplo:** Dados alimentando um app

**Você trabalha com:**
- Backend simples (API)
- Autenticação
- Queries analíticas
- Consumo de dados

**Você aprende:**
- Dados como produto
- Interface entre engenharia e negócio
- Exposição segura de dados
- Pensamento full-stack data

---

## 9. Pipeline com Processamento Distribuído
**Nível:** Intermediário/Avançado | **Complexidade:** ⭐⭐⭐⭐

**Exemplo:** Spotify + Spark / Databricks

**Você trabalha com:**
- Spark (PySpark)
- Databricks
- Delta Lake
- Partitioning, caching, joins

**Você aprende:**
- Diferença entre pandas × Spark
- Processamento em larga escala
- Performance e custo
- Arquitetura Lakehouse
- Quando usar processamento distribuído vs. processamento local

---

## 10. Projeto com Infraestrutura como Código
**Nível:** Intermediário/Avançado | **Complexidade:** ⭐⭐⭐⭐

**Exemplo:** Pipeline 100% reproduzível

**Você trabalha com:**
- Docker
- Terraform
- Cloud (AWS/GCP/Azure)
- CI/CD

**Você aprende:**
- Ambientes reproduzíveis
- Deploy de pipelines
- Infraestrutura moderna
- Engenharia além do código
- DevOps para engenharia de dados

---

## 11. Data Catalog e Metadata Management
**Nível:** Intermediário/Avançado | **Complexidade:** ⭐⭐⭐⭐
**(A IA sugeriu isso)**

**Exemplo:** Sistema para documentar, catalogar e descobrir datasets

**Você trabalha com:**
- OpenMetadata / DataHub / Amundsen
- Metadados estruturados
- APIs de metadados
- Linhagem de dados básica

**Você aprende:**
- Governança de dados
- Descoberta de dados
- Documentação como código
- Colaboração entre times
- **Por que é importante:** Em empresas grandes, encontrar e entender dados é um desafio real

---

## 12. Projeto de Streaming / Near Real-Time
**Nível:** Avançado | **Complexidade:** ⭐⭐⭐⭐⭐

**Exemplo:** Eventos, logs, mensagens

**Você trabalha com:**
- Spark Structured Streaming
- Kafka / PubSub / Kinesis
- Janelas de tempo
- Watermarks

**Você aprende:**
- Batch vs streaming
- Latência vs throughput
- Processamento contínuo
- Design de pipelines em tempo real
- Trade-offs entre batch e streaming

---

## 13. Engine de Geração Dinâmica de SQL
**Nível:** Avançado | **Complexidade:** ⭐⭐⭐⭐⭐

**Exemplo:** SQL criado a partir de metadados

**Você trabalha com:**
- Python avançado
- Metadados
- Templates SQL
- Abstração de lógica

**Você aprende:**
- Automação de engenharia
- Redução de código repetido
- Design de frameworks internos
- Pensar como engenheira sênior
- DRY (Don't Repeat Yourself) aplicado a pipelines

---

## 14. Data Lineage e Observability Completa
**Nível:** Avançado | **Complexidade:** ⭐⭐⭐⭐⭐
**(A IA sugeriu isso)**

**Exemplo:** Rastrear impacto de mudanças, monitorar saúde de pipelines, alertas inteligentes

**Você trabalha com:**
- OpenLineage / DataHub
- Métricas e logs estruturados
- Grafana / Prometheus
- Alertas baseados em ML

**Você aprende:**
- Rastreabilidade completa de dados
- Impact analysis (o que quebra se eu mudar isso?)
- Observability vs. monitoring
- Debugging de pipelines complexos
- **Por que é importante:** Em sistemas complexos, entender o impacto de mudanças é crítico

---

## 15. Sistema de Feature Store
**Nível:** Avançado | **Complexidade:** ⭐⭐⭐⭐⭐
**(A IA sugeriu isso)**

**Exemplo:** Armazenar e servir features para modelos de ML

**Você trabalha com:**
- Feast / Tecton / Hopsworks
- Versionamento de features
- APIs de feature serving
- Integração com ML pipelines

**Você aprende:**
- MLOps e engenharia de dados para ML
- Reutilização de features
- Consistência entre treino e inferência
- **Por que é importante:** Conecta engenharia de dados com ciência de dados de forma profissional

---

## Guia de Progressão de Aprendizado

### Fase 1: Fundamentos (Projetos 1-3)
**Objetivo:** Entender o básico de ETL e pipelines
- Comece com arquivos locais
- Evolua para APIs e bancos de dados
- Aprenda a validar qualidade desde o início

### Fase 2: Produção (Projetos 4-6)
**Objetivo:** Pipelines robustos e arquiteturas escaláveis
- Orquestração profissional
- Migrações e arquiteturas corporativas
- Padrões de indústria (Medallion Architecture)

### Fase 3: Especialização (Projetos 7-10)
**Objetivo:** Técnicas avançadas e DevOps
- CDC, processamento distribuído
- Infraestrutura como código
- Integração com aplicações

### Fase 4: Excelência (Projetos 11-15)
**Objetivo:** Engenharia de dados de nível sênior
- Governança, observability
- Streaming em tempo real
- Frameworks e abstrações
- Integração com ML

---

## Dicas do Especialista

1. **Não pule etapas:** Cada projeto constrói sobre os anteriores
2. **Faça projetos reais:** Use dados públicos interessantes (ex: dados do governo, APIs abertas)
3. **Documente tudo:** Aprenda a documentar como se outros engenheiros fossem usar seu código
4. **Pense em produção:** Sempre considere: "E se isso rodar em produção com milhões de linhas?"
5. **Aprenda SQL profundamente:** É a linguagem mais importante para engenharia de dados
6. **Entenda o negócio:** Os melhores engenheiros de dados entendem o problema que estão resolvendo
