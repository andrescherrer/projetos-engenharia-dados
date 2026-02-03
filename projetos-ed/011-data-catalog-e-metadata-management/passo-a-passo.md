# Passo a Passo: Data Catalog e Metadata Management

## Objetivo
Implementar um sistema de catálogo de dados para documentar, catalogar e facilitar a descoberta de datasets, melhorando a governança de dados.

## Pré-requisitos
- Docker e Docker Compose
- Python 3.8+
- Conhecimento básico de APIs REST

## Passo 1: Escolher Ferramenta

**Opções:**
- **OpenMetadata:** Moderno, open-source, fácil de usar
- **DataHub:** LinkedIn, muito completo
- **Amundsen:** Lyft, focado em descoberta

Vamos usar **OpenMetadata** (mais simples para começar).

## Passo 2: Estrutura do Projeto

```
data-catalog/
├── docker-compose.yml
├── src/
│   ├── collectors/
│   │   └── metadata_collector.py
│   └── api/
│       └── catalog_api.py
└── metadata/
    └── schemas/
```

## Passo 3: Configurar OpenMetadata

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  mysql:
    image: openmetadata/db:latest
    environment:
      MYSQL_ROOT_PASSWORD: openmetadata_password
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.2
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  openmetadata-server:
    image: openmetadata/server:latest
    depends_on:
      - mysql
      - elasticsearch
    environment:
      OPENMETADATA_AIRFLOW_APIS_ENABLED: "true"
    ports:
      - "8585:8585"
    volumes:
      - ./metadata:/metadata

volumes:
  mysql-data:
  elasticsearch-data:
```

## Passo 4: Criar Coletor de Metadados

**src/collectors/metadata_collector.py:**
```python
from openmetadata import OpenMetadata
from openmetadata.ingestion.api.source import Source, SourceStatus
from openmetadata.ingestion.models.table_metadata import TableMetadata
from typing import Iterator, List
import json

class DatabaseMetadataCollector:
    """Coleta metadados de banco de dados"""
    
    def __init__(self, connection_string: str, database_name: str):
        self.connection_string = connection_string
        self.database_name = database_name
        self.metadata_client = OpenMetadata()
    
    def collect_table_metadata(self, schema: str, table: str) -> dict:
        """Coleta metadados de uma tabela"""
        from sqlalchemy import create_engine, inspect
        
        engine = create_engine(self.connection_string)
        inspector = inspect(engine)
        
        # Obter colunas
        columns = inspector.get_columns(table, schema=schema)
        
        # Obter índices
        indexes = inspector.get_indexes(table, schema=schema)
        
        # Obter foreign keys
        foreign_keys = inspector.get_foreign_keys(table, schema=schema)
        
        metadata = {
            "database": self.database_name,
            "schema": schema,
            "table": table,
            "columns": [
                {
                    "name": col["name"],
                    "type": str(col["type"]),
                    "nullable": col.get("nullable", True),
                    "default": str(col.get("default", ""))
                }
                for col in columns
            ],
            "indexes": [
                {
                    "name": idx["name"],
                    "columns": idx["column_names"],
                    "unique": idx.get("unique", False)
                }
                for idx in indexes
            ],
            "foreign_keys": [
                {
                    "name": fk["name"],
                    "constrained_columns": fk["constrained_columns"],
                    "referred_table": fk["referred_table"],
                    "referred_columns": fk["referred_columns"]
                }
                for fk in foreign_keys
            ]
        }
        
        return metadata
    
    def collect_all_tables(self, schema: str = None) -> List[dict]:
        """Coleta metadados de todas as tabelas"""
        from sqlalchemy import create_engine, inspect
        
        engine = create_engine(self.connection_string)
        inspector = inspect(engine)
        
        schemas = [schema] if schema else inspector.get_schema_names()
        all_metadata = []
        
        for schema_name in schemas:
            tables = inspector.get_table_names(schema=schema_name)
            
            for table_name in tables:
                print(f"🔄 Coletando metadados: {schema_name}.{table_name}")
                metadata = self.collect_table_metadata(schema_name, table_name)
                all_metadata.append(metadata)
        
        return all_metadata
    
    def publish_to_catalog(self, metadata: dict):
        """Publica metadados no catálogo"""
        # Criar tabela no OpenMetadata
        table_metadata = TableMetadata(
            name=metadata["table"],
            database=metadata["database"],
            schema=metadata["schema"],
            columns=metadata["columns"]
        )
        
        self.metadata_client.create_or_update(table_metadata)
        print(f"✅ Metadados publicados: {metadata['schema']}.{metadata['table']}")

# Exemplo de uso
if __name__ == "__main__":
    collector = DatabaseMetadataCollector(
        connection_string="postgresql://user:pass@localhost/analytics_db",
        database_name="analytics_db"
    )
    
    all_metadata = collector.collect_all_tables(schema="analytics")
    
    for metadata in all_metadata:
        collector.publish_to_catalog(metadata)
```

## Passo 5: Criar API de Catálogo

**src/api/catalog_api.py:**
```python
from fastapi import FastAPI, Query
from typing import List, Optional
from pydantic import BaseModel
from openmetadata import OpenMetadata

app = FastAPI(title="Data Catalog API")

class TableInfo(BaseModel):
    database: str
    schema: str
    table: str
    columns: List[dict]
    description: Optional[str] = None

class SearchResult(BaseModel):
    table: str
    schema: str
    database: str
    match_score: float

metadata_client = OpenMetadata()

@app.get("/api/catalog/tables", response_model=List[TableInfo])
async def list_tables(
    database: Optional[str] = Query(None),
    schema: Optional[str] = Query(None)
):
    """Lista todas as tabelas no catálogo"""
    filters = {}
    if database:
        filters["database"] = database
    if schema:
        filters["schema"] = schema
    
    tables = metadata_client.list_tables(**filters)
    return [TableInfo(**table.dict()) for table in tables]

@app.get("/api/catalog/search")
async def search_tables(
    query: str = Query(..., description="Termo de busca"),
    limit: int = Query(10, le=100)
) -> List[SearchResult]:
    """Busca tabelas por termo"""
    results = metadata_client.search_tables(query, limit=limit)
    
    return [
        SearchResult(
            table=result["table"],
            schema=result["schema"],
            database=result["database"],
            match_score=result.get("score", 0.0)
        )
        for result in results
    ]

@app.get("/api/catalog/table/{database}/{schema}/{table}")
async def get_table_details(
    database: str,
    schema: str,
    table: str
) -> TableInfo:
    """Obtém detalhes de uma tabela específica"""
    table_info = metadata_client.get_table(database, schema, table)
    return TableInfo(**table_info.dict())

@app.post("/api/catalog/table/{database}/{schema}/{table}/description")
async def update_table_description(
    database: str,
    schema: str,
    table: str,
    description: str
):
    """Atualiza descrição de uma tabela"""
    metadata_client.update_table_description(
        database, schema, table, description
    )
    return {"status": "updated"}

@app.get("/api/catalog/lineage/{database}/{schema}/{table}")
async def get_table_lineage(
    database: str,
    schema: str,
    table: str
):
    """Obtém linhagem de uma tabela"""
    lineage = metadata_client.get_lineage(database, schema, table)
    return lineage

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Passo 6: Documentação como Código

**metadata/schemas/analytics_schema.yaml:**
```yaml
database: analytics_db
schema: analytics
tables:
  - name: fact_vendas
    description: |
      Tabela fato contendo transações de vendas.
      Fonte: Sistema ERP
      Atualização: Diária
    columns:
      - name: fact_id
        description: Identificador único da transação
        data_type: integer
        tags: [primary_key, identifier]
      - name: date_id
        description: Referência para dimensão de data
        data_type: integer
        tags: [foreign_key, date]
      - name: customer_id
        description: Referência para dimensão de cliente
        data_type: integer
        tags: [foreign_key, customer]
      - name: valor_total
        description: Valor total da venda em reais
        data_type: decimal
        tags: [metric, financial]
    owners:
      - email: data-team@company.com
    tags: [sales, fact, star-schema]
    
  - name: dim_customer
    description: |
      Dimensão de clientes com SCD Type 2.
      Mantém histórico de mudanças.
    columns:
      - name: customer_id
        description: Identificador único do cliente
        data_type: integer
      - name: nome
        description: Nome completo do cliente
        data_type: varchar
      - name: cidade
        description: Cidade de residência
        data_type: varchar
    tags: [dimension, customer, scd-type-2]
```

## Passo 7: Script de Coleta Automática

**scripts/collect_metadata.sh:**
```bash
#!/bin/bash

echo "🔄 Coletando metadados..."

# Coletar de múltiplas fontes
python src/collectors/metadata_collector.py \
  --source postgresql://user:pass@localhost/analytics_db \
  --database analytics_db \
  --schema analytics

python src/collectors/metadata_collector.py \
  --source postgresql://user:pass@localhost/warehouse_db \
  --database warehouse_db \
  --schema public

echo "✅ Metadados coletados e publicados no catálogo"
```

## Passo 8: Integração com Pipeline

**src/integration/pipeline_integration.py:**
```python
from openmetadata import OpenMetadata
from datetime import datetime

class PipelineMetadataTracker:
    """Rastreia metadados durante execução do pipeline"""
    
    def __init__(self):
        self.metadata_client = OpenMetadata()
    
    def log_pipeline_start(self, pipeline_name: str, source: str, target: str):
        """Registra início do pipeline"""
        self.metadata_client.create_pipeline_run(
            pipeline_name=pipeline_name,
            source=source,
            target=target,
            start_time=datetime.now(),
            status="running"
        )
    
    def log_pipeline_end(self, pipeline_name: str, status: str, rows_processed: int):
        """Registra fim do pipeline"""
        self.metadata_client.update_pipeline_run(
            pipeline_name=pipeline_name,
            end_time=datetime.now(),
            status=status,
            rows_processed=rows_processed
        )
    
    def update_table_stats(self, database: str, schema: str, table: str, stats: dict):
        """Atualiza estatísticas de uma tabela"""
        self.metadata_client.update_table_stats(
            database=database,
            schema=schema,
            table=table,
            row_count=stats.get("row_count"),
            size_bytes=stats.get("size_bytes"),
            last_updated=datetime.now()
        )

# Uso no pipeline
def run_pipeline():
    tracker = PipelineMetadataTracker()
    
    tracker.log_pipeline_start(
        pipeline_name="daily_sales_etl",
        source="erp_db.vendas",
        target="analytics_db.fact_vendas"
    )
    
    try:
        # Executar pipeline...
        rows_processed = 10000
        
        tracker.log_pipeline_end(
            pipeline_name="daily_sales_etl",
            status="success",
            rows_processed=rows_processed
        )
        
        tracker.update_table_stats(
            database="analytics_db",
            schema="analytics",
            table="fact_vendas",
            stats={"row_count": rows_processed}
        )
    except Exception as e:
        tracker.log_pipeline_end(
            pipeline_name="daily_sales_etl",
            status="failed",
            rows_processed=0
        )
        raise
```

## Checklist de Conclusão

- [ ] OpenMetadata configurado
- [ ] Coletor de metadados implementado
- [ ] API de catálogo criada
- [ ] Documentação como código
- [ ] Integração com pipeline
- [ ] Busca funcionando
- [ ] Linhagem de dados implementada
- [ ] Documentação completa
