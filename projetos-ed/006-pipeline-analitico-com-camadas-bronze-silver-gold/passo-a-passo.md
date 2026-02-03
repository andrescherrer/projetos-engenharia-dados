# Passo a Passo: Pipeline Analítico com Camadas (Bronze / Silver / Gold)

## Objetivo
Implementar a Medallion Architecture (Bronze/Silver/Gold) para organizar dados em camadas progressivamente refinadas, otimizando para consumo analítico.

## Pré-requisitos
- Python 3.8+
- Spark ou Delta Lake (Databricks, Spark local)
- Conhecimento de SQL e processamento de dados

## Passo 1: Entender Medallion Architecture

**Bronze (Raw Layer):**
- Dados brutos como recebidos da fonte
- Sem transformações
- Histórico completo preservado
- Formato: Parquet, Delta, JSON

**Silver (Cleaned Layer):**
- Dados limpos e validados
- Schema aplicado
- Duplicatas removidas
- Pronto para transformações

**Gold (Business Layer):**
- Dados agregados e modelados
- Otimizado para consumo
- Métricas de negócio
- Pronto para dashboards/relatórios

## Passo 2: Configurar Ambiente

```bash
# Instalar dependências
pip install pyspark delta-spark pandas

# Para Databricks, use o runtime que já inclui Delta Lake
```

## Passo 3: Estrutura do Projeto

```
medallion-pipeline/
├── src/
│   ├── bronze/
│   │   └── ingest.py
│   ├── silver/
│   │   └── clean.py
│   ├── gold/
│   │   └── aggregate.py
│   └── utils/
│       └── spark_utils.py
├── notebooks/
│   ├── 01_bronze_ingestion.ipynb
│   ├── 02_silver_cleaning.ipynb
│   └── 03_gold_aggregation.ipynb
└── config/
    └── paths.py
```

## Passo 4: Configurar Spark com Delta Lake

**src/utils/spark_utils.py:**
```python
from pyspark.sql import SparkSession
from delta import configure_spark_with_delta_pip

def create_spark_session(app_name: str = "Medallion Pipeline"):
    """Cria SparkSession configurado com Delta Lake"""
    
    builder = SparkSession.builder \
        .appName(app_name) \
        .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
        .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    
    spark = configure_spark_with_delta_pip(builder).getOrCreate()
    
    return spark

def get_bronze_path(table_name: str, base_path: str = "data/bronze") -> str:
    """Retorna caminho da tabela Bronze"""
    return f"{base_path}/{table_name}"

def get_silver_path(table_name: str, base_path: str = "data/silver") -> str:
    """Retorna caminho da tabela Silver"""
    return f"{base_path}/{table_name}"

def get_gold_path(table_name: str, base_path: str = "data/gold") -> str:
    """Retorna caminho da tabela Gold"""
    return f"{base_path}/{table_name}"
```

## Passo 5: Implementar Camada Bronze (Ingestão)

**src/bronze/ingest.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import current_timestamp, col, input_file_name
from delta.tables import DeltaTable
from src.utils.spark_utils import create_spark_session, get_bronze_path

class BronzeIngestion:
    """Ingere dados brutos na camada Bronze"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def ingest_from_csv(self, source_path: str, table_name: str):
        """Ingere dados de CSV para Bronze"""
        print(f"🔄 Ingestando {source_path} para Bronze/{table_name}")
        
        # Ler CSV
        df = self.spark.read \
            .option("header", "true") \
            .option("inferSchema", "true") \
            .csv(source_path)
        
        # Adicionar metadados
        df = df.withColumn("ingestion_timestamp", current_timestamp()) \
               .withColumn("source_file", input_file_name())
        
        # Salvar como Delta Table
        bronze_path = get_bronze_path(table_name)
        df.write \
            .format("delta") \
            .mode("append") \
            .option("mergeSchema", "true") \
            .save(bronze_path)
        
        print(f"✅ Dados ingeridos em {bronze_path}")
        return bronze_path
    
    def ingest_from_json(self, source_path: str, table_name: str):
        """Ingere dados de JSON para Bronze"""
        print(f"🔄 Ingestando {source_path} para Bronze/{table_name}")
        
        df = self.spark.read \
            .option("multiline", "true") \
            .json(source_path)
        
        df = df.withColumn("ingestion_timestamp", current_timestamp()) \
               .withColumn("source_file", input_file_name())
        
        bronze_path = get_bronze_path(table_name)
        df.write \
            .format("delta") \
            .mode("append") \
            .option("mergeSchema", "true") \
            .save(bronze_path)
        
        print(f"✅ Dados ingeridos em {bronze_path}")
        return bronze_path
    
    def ingest_from_api(self, api_url: str, table_name: str):
        """Ingere dados de API para Bronze"""
        import requests
        import json
        from pyspark.sql.types import StructType
        
        print(f"🔄 Ingestando dados de API para Bronze/{table_name}")
        
        response = requests.get(api_url)
        data = response.json()
        
        # Converter para DataFrame
        df = self.spark.createDataFrame(data)
        df = df.withColumn("ingestion_timestamp", current_timestamp())
        
        bronze_path = get_bronze_path(table_name)
        df.write \
            .format("delta") \
            .mode("append") \
            .option("mergeSchema", "true") \
            .save(bronze_path)
        
        print(f"✅ Dados ingeridos em {bronze_path}")
        return bronze_path

# Exemplo de uso
if __name__ == "__main__":
    spark = create_spark_session()
    ingester = BronzeIngestion(spark)
    
    # Ingestar CSV
    ingester.ingest_from_csv("data/raw/vendas.csv", "vendas")
```

## Passo 6: Implementar Camada Silver (Limpeza)

**src/silver/clean.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, trim, upper, to_date, current_timestamp
from pyspark.sql.types import StringType, IntegerType, DoubleType, DateType
from delta.tables import DeltaTable
from src.utils.spark_utils import get_bronze_path, get_silver_path

class SilverCleaning:
    """Limpa e valida dados na camada Silver"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def clean_bronze_table(self, bronze_table: str, silver_table: str, schema: dict):
        """Lê Bronze, limpa e salva em Silver"""
        print(f"🔄 Processando Bronze/{bronze_table} → Silver/{silver_table}")
        
        # Ler Bronze
        bronze_path = get_bronze_path(bronze_table)
        df = self.spark.read.format("delta").load(bronze_path)
        
        # Aplicar schema
        df_cleaned = self._apply_schema(df, schema)
        
        # Limpar dados
        df_cleaned = self._clean_data(df_cleaned)
        
        # Remover duplicatas
        df_cleaned = df_cleaned.dropDuplicates()
        
        # Adicionar metadados
        df_cleaned = df_cleaned.withColumn("cleaned_timestamp", current_timestamp())
        
        # Salvar em Silver
        silver_path = get_silver_path(silver_table)
        df_cleaned.write \
            .format("delta") \
            .mode("overwrite") \
            .option("overwriteSchema", "true") \
            .save(silver_path)
        
        print(f"✅ Dados limpos salvos em {silver_path}")
        return silver_path
    
    def _apply_schema(self, df, schema: dict):
        """Aplica schema definido"""
        for col_name, col_type in schema.items():
            if col_name in df.columns:
                if col_type == "string":
                    df = df.withColumn(col_name, col(col_name).cast(StringType()))
                elif col_type == "int":
                    df = df.withColumn(col_name, col(col_name).cast(IntegerType()))
                elif col_type == "double":
                    df = df.withColumn(col_name, col(col_name).cast(DoubleType()))
                elif col_type == "date":
                    df = df.withColumn(col_name, to_date(col(col_name)))
        
        return df
    
    def _clean_data(self, df):
        """Aplica limpezas comuns"""
        # Trim strings
        for col_name in df.columns:
            if df.schema[col_name].dataType == StringType():
                df = df.withColumn(col_name, trim(col(col_name)))
        
        # Padronizar maiúsculas/minúsculas em colunas específicas
        # Exemplo: categorias sempre em maiúscula
        if "categoria" in df.columns:
            df = df.withColumn("categoria", upper(col("categoria")))
        
        # Validar valores
        # Exemplo: valores negativos não permitidos
        if "valor" in df.columns:
            df = df.withColumn(
                "valor",
                when(col("valor") < 0, None).otherwise(col("valor"))
            )
        
        return df
    
    def incremental_silver_update(self, bronze_table: str, silver_table: str, 
                                  key_column: str = "id"):
        """Atualização incremental usando merge"""
        print(f"🔄 Atualização incremental: Bronze/{bronze_table} → Silver/{silver_table}")
        
        bronze_path = get_bronze_path(bronze_table)
        silver_path = get_silver_path(silver_table)
        
        # Ler dados novos do Bronze
        bronze_df = self.spark.read.format("delta").load(bronze_path)
        
        # Ler Silver existente
        try:
            silver_delta = DeltaTable.forPath(self.spark, silver_path)
            
            # Merge: atualizar existentes, inserir novos
            silver_delta.alias("silver") \
                .merge(
                    bronze_df.alias("bronze"),
                    f"silver.{key_column} = bronze.{key_column}"
                ) \
                .whenMatchedUpdateAll() \
                .whenNotMatchedInsertAll() \
                .execute()
            
            print("✅ Merge incremental concluído")
        except:
            # Primeira execução: criar tabela
            bronze_df.write \
                .format("delta") \
                .mode("overwrite") \
                .save(silver_path)
            print("✅ Tabela Silver criada")

# Exemplo de uso
if __name__ == "__main__":
    from src.utils.spark_utils import create_spark_session
    
    spark = create_spark_session()
    cleaner = SilverCleaning(spark)
    
    schema = {
        "id": "int",
        "nome": "string",
        "valor": "double",
        "data": "date"
    }
    
    cleaner.clean_bronze_table("vendas", "vendas_cleaned", schema)
```

## Passo 7: Implementar Camada Gold (Agregações)

**src/gold/aggregate.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import sum, avg, count, col, year, month, current_timestamp
from pyspark.sql import Window
from src.utils.spark_utils import get_silver_path, get_gold_path

class GoldAggregation:
    """Cria agregações e métricas na camada Gold"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def create_sales_summary(self, silver_table: str = "vendas_cleaned"):
        """Cria resumo de vendas"""
        print(f"🔄 Criando resumo de vendas em Gold")
        
        # Ler Silver
        silver_path = get_silver_path(silver_table)
        df = self.spark.read.format("delta").load(silver_path)
        
        # Agregações por data
        sales_summary = df.groupBy(
            year("data").alias("ano"),
            month("data").alias("mes"),
            "categoria"
        ).agg(
            sum("valor").alias("total_vendas"),
            avg("valor").alias("media_vendas"),
            count("*").alias("quantidade_vendas")
        ).withColumn("created_at", current_timestamp())
        
        # Salvar em Gold
        gold_path = get_gold_path("sales_summary")
        sales_summary.write \
            .format("delta") \
            .mode("overwrite") \
            .save(gold_path)
        
        print(f"✅ Resumo de vendas salvo em {gold_path}")
        return gold_path
    
    def create_customer_metrics(self, silver_table: str = "vendas_cleaned"):
        """Cria métricas por cliente"""
        print(f"🔄 Criando métricas de clientes em Gold")
        
        silver_path = get_silver_path(silver_table)
        df = self.spark.read.format("delta").load(silver_path)
        
        # Métricas por cliente
        customer_metrics = df.groupBy("cliente_id").agg(
            sum("valor").alias("total_gasto"),
            count("*").alias("total_compras"),
            avg("valor").alias("ticket_medio"),
            max("data").alias("ultima_compra")
        ).withColumn("created_at", current_timestamp())
        
        gold_path = get_gold_path("customer_metrics")
        customer_metrics.write \
            .format("delta") \
            .mode("overwrite") \
            .save(gold_path)
        
        print(f"✅ Métricas de clientes salvas em {gold_path}")
        return gold_path
    
    def create_time_series(self, silver_table: str = "vendas_cleaned", 
                          date_column: str = "data", value_column: str = "valor"):
        """Cria série temporal agregada"""
        print(f"🔄 Criando série temporal em Gold")
        
        silver_path = get_silver_path(silver_table)
        df = self.spark.read.format("delta").load(silver_path)
        
        # Agregação diária
        time_series = df.groupBy(date_column).agg(
            sum(value_column).alias("valor_total"),
            count("*").alias("quantidade"),
            avg(value_column).alias("valor_medio")
        ).orderBy(date_column)
        
        gold_path = get_gold_path("time_series")
        time_series.write \
            .format("delta") \
            .mode("overwrite") \
            .save(gold_path)
        
        print(f"✅ Série temporal salva em {gold_path}")
        return gold_path

# Exemplo de uso
if __name__ == "__main__":
    from src.utils.spark_utils import create_spark_session
    
    spark = create_spark_session()
    aggregator = GoldAggregation(spark)
    
    aggregator.create_sales_summary()
    aggregator.create_customer_metrics()
    aggregator.create_time_series()
```

## Passo 8: Pipeline Completo

**main.py:**
```python
from src.utils.spark_utils import create_spark_session
from src.bronze.ingest import BronzeIngestion
from src.silver.clean import SilverCleaning
from src.gold.aggregate import GoldAggregation

def main():
    # Inicializar Spark
    spark = create_spark_session("Medallion Pipeline")
    
    print("=" * 60)
    print("PIPELINE MEDALLION ARCHITECTURE")
    print("=" * 60)
    
    # 1. Bronze: Ingestão
    print("\n📥 CAMADA BRONZE: Ingestão")
    ingester = BronzeIngestion(spark)
    ingester.ingest_from_csv("data/raw/vendas.csv", "vendas")
    
    # 2. Silver: Limpeza
    print("\n🧹 CAMADA SILVER: Limpeza")
    cleaner = SilverCleaning(spark)
    schema = {
        "id": "int",
        "cliente_id": "int",
        "produto_id": "int",
        "valor": "double",
        "quantidade": "int",
        "data": "date",
        "categoria": "string"
    }
    cleaner.clean_bronze_table("vendas", "vendas_cleaned", schema)
    
    # 3. Gold: Agregações
    print("\n✨ CAMADA GOLD: Agregações")
    aggregator = GoldAggregation(spark)
    aggregator.create_sales_summary()
    aggregator.create_customer_metrics()
    aggregator.create_time_series()
    
    print("\n✅ Pipeline Medallion concluído!")
    spark.stop()

if __name__ == "__main__":
    main()
```

## Passo 9: Otimizações Delta Lake

**src/utils/optimize.py:**
```python
from delta.tables import DeltaTable
from pyspark.sql import SparkSession

def optimize_delta_table(spark: SparkSession, table_path: str):
    """Otimiza tabela Delta"""
    delta_table = DeltaTable.forPath(spark, table_path)
    
    # Compactar arquivos pequenos
    delta_table.optimize().executeCompaction()
    
    # Limpar arquivos antigos
    delta_table.vacuum(retentionHours=168)  # 7 dias
    
    print(f"✅ Tabela {table_path} otimizada")

def z_order_table(spark: SparkSession, table_path: str, columns: list):
    """Aplica Z-Order para melhorar performance de queries"""
    delta_table = DeltaTable.forPath(spark, table_path)
    
    delta_table.optimize().executeZOrder(columns)
    
    print(f"✅ Z-Order aplicado em {columns}")
```

## Passo 10: Consultas Analíticas

**queries/analytical_queries.sql:**
```sql
-- Consultar Gold Layer (otimizado para análise)
SELECT 
    ano,
    mes,
    categoria,
    total_vendas,
    quantidade_vendas
FROM delta.`data/gold/sales_summary`
WHERE ano = 2024
ORDER BY total_vendas DESC;

-- Comparar períodos
SELECT 
    categoria,
    SUM(CASE WHEN ano = 2023 THEN total_vendas ELSE 0 END) as vendas_2023,
    SUM(CASE WHEN ano = 2024 THEN total_vendas ELSE 0 END) as vendas_2024,
    (SUM(CASE WHEN ano = 2024 THEN total_vendas ELSE 0 END) - 
     SUM(CASE WHEN ano = 2023 THEN total_vendas ELSE 0 END)) / 
     SUM(CASE WHEN ano = 2023 THEN total_vendas ELSE 0 END) * 100 as crescimento_pct
FROM delta.`data/gold/sales_summary`
GROUP BY categoria;
```

## Checklist de Conclusão

- [ ] Ambiente Spark/Delta configurado
- [ ] Camada Bronze implementada
- [ ] Camada Silver implementada
- [ ] Camada Gold implementada
- [ ] Pipeline completo funcionando
- [ ] Otimizações Delta aplicadas
- [ ] Queries analíticas testadas
- [ ] Documentação completa
