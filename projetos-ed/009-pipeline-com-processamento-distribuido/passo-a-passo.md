# Passo a Passo: Pipeline com Processamento Distribuído

## Objetivo
Implementar um pipeline usando Spark para processamento distribuído de grandes volumes de dados, aprendendo quando e como usar processamento distribuído.

## Pré-requisitos
- Python 3.8+
- Java 8+ (requerido pelo Spark)
- Conhecimento básico de Spark/PySpark

## Passo 1: Quando Usar Processamento Distribuído

**Use Spark quando:**
- Dados não cabem na memória de uma máquina
- Processamento leva muito tempo (>30min)
- Precisa processar terabytes de dados
- Precisa de paralelismo massivo

**Use Pandas quando:**
- Dados cabem na memória (<10GB)
- Processamento é rápido (<5min)
- Lógica é complexa e difícil de paralelizar

## Passo 2: Configurar Spark Local

```bash
# Instalar PySpark
pip install pyspark

# Verificar instalação
python -c "from pyspark.sql import SparkSession; print('Spark OK')"
```

## Passo 3: Estrutura do Projeto

```
spark-pipeline/
├── src/
│   ├── extract/
│   │   └── spark_extract.py
│   ├── transform/
│   │   └── spark_transform.py
│   ├── load/
│   │   └── spark_load.py
│   └── utils/
│       └── spark_config.py
├── data/
│   └── input/
└── notebooks/
    └── spark_analysis.ipynb
```

## Passo 4: Configurar Spark Session

**src/utils/spark_config.py:**
```python
from pyspark.sql import SparkSession
from pyspark import SparkConf

def create_spark_session(
    app_name: str = "Data Pipeline",
    master: str = "local[*]",
    memory: str = "4g"
) -> SparkSession:
    """Cria SparkSession otimizada"""
    
    conf = SparkConf().setAppName(app_name) \
        .setMaster(master) \
        .set("spark.executor.memory", memory) \
        .set("spark.driver.memory", memory) \
        .set("spark.sql.shuffle.partitions", "200") \
        .set("spark.default.parallelism", "200") \
        .set("spark.sql.adaptive.enabled", "true") \
        .set("spark.sql.adaptive.coalescePartitions.enabled", "true")
    
    spark = SparkSession.builder \
        .config(conf=conf) \
        .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    return spark
```

## Passo 5: Extração com Spark

**src/extract/spark_extract.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, input_file_name, current_timestamp

class SparkExtractor:
    """Extrai dados usando Spark"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def extract_csv(self, path: str, header: bool = True):
        """Extrai CSV distribuído"""
        print(f"🔄 Extraindo CSV: {path}")
        
        df = self.spark.read \
            .option("header", str(header).lower()) \
            .option("inferSchema", "true") \
            .csv(path)
        
        # Adicionar metadados
        df = df.withColumn("source_file", input_file_name()) \
               .withColumn("extracted_at", current_timestamp())
        
        print(f"✅ Extraído: {df.count()} registros, {len(df.columns)} colunas")
        return df
    
    def extract_parquet(self, path: str):
        """Extrai Parquet (formato otimizado)"""
        print(f"🔄 Extraindo Parquet: {path}")
        
        df = self.spark.read.parquet(path)
        print(f"✅ Extraído: {df.count()} registros")
        return df
    
    def extract_json(self, path: str, multiline: bool = True):
        """Extrai JSON"""
        print(f"🔄 Extraindo JSON: {path}")
        
        df = self.spark.read \
            .option("multiline", str(multiline).lower()) \
            .json(path)
        
        print(f"✅ Extraído: {df.count()} registros")
        return df
    
    def extract_from_database(self, jdbc_url: str, table: str, 
                             user: str, password: str):
        """Extrai de banco de dados"""
        print(f"🔄 Extraindo do banco: {table}")
        
        df = self.spark.read \
            .format("jdbc") \
            .option("url", jdbc_url) \
            .option("dbtable", table) \
            .option("user", user) \
            .option("password", password) \
            .option("numPartitions", "10") \
            .option("partitionColumn", "id") \
            .option("lowerBound", "1") \
            .option("upperBound", "1000000") \
            .load()
        
        print(f"✅ Extraído: {df.count()} registros")
        return df
```

## Passo 6: Transformações com Spark

**src/transform/spark_transform.py:**
```python
from pyspark.sql import SparkSession, DataFrame
from pyspark.sql.functions import (
    col, when, sum, avg, count, max, min,
    date_format, year, month, dayofmonth,
    regexp_replace, trim, upper, lower
)
from pyspark.sql.window import Window

class SparkTransformer:
    """Aplica transformações distribuídas"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def clean_data(self, df: DataFrame) -> DataFrame:
        """Limpa dados"""
        print("🔄 Limpando dados...")
        
        # Remover duplicatas
        df_clean = df.dropDuplicates()
        
        # Trim strings
        for col_name in df_clean.columns:
            if df_clean.schema[col_name].dataType.typeName() == 'string':
                df_clean = df_clean.withColumn(
                    col_name,
                    trim(col(col_name))
                )
        
        print(f"✅ Dados limpos: {df_clean.count()} registros")
        return df_clean
    
    def aggregate_data(
        self,
        df: DataFrame,
        group_by_cols: list,
        agg_dict: dict
    ) -> DataFrame:
        """Agrega dados"""
        print(f"🔄 Agregando por: {group_by_cols}")
        
        # Construir expressões de agregação
        agg_exprs = []
        for col_name, func in agg_dict.items():
            if func == 'sum':
                agg_exprs.append(sum(col(col_name)).alias(f"total_{col_name}"))
            elif func == 'avg':
                agg_exprs.append(avg(col(col_name)).alias(f"avg_{col_name}"))
            elif func == 'count':
                agg_exprs.append(count(col(col_name)).alias(f"count_{col_name}"))
            elif func == 'max':
                agg_exprs.append(max(col(col_name)).alias(f"max_{col_name}"))
            elif func == 'min':
                agg_exprs.append(min(col(col_name)).alias(f"min_{col_name}"))
        
        df_agg = df.groupBy(*group_by_cols).agg(*agg_exprs)
        
        print(f"✅ Agregação concluída: {df_agg.count()} grupos")
        return df_agg
    
    def join_dataframes(
        self,
        df1: DataFrame,
        df2: DataFrame,
        join_key: str,
        join_type: str = "inner"
    ) -> DataFrame:
        """Faz join de DataFrames"""
        print(f"🔄 Fazendo join {join_type} por {join_key}")
        
        df_joined = df1.join(df2, on=join_key, how=join_type)
        
        print(f"✅ Join concluído: {df_joined.count()} registros")
        return df_joined
    
    def window_aggregation(
        self,
        df: DataFrame,
        partition_by: list,
        order_by: str,
        window_func: str = "sum",
        value_col: str = "valor"
    ) -> DataFrame:
        """Aplica funções de janela"""
        print(f"🔄 Aplicando window function")
        
        window_spec = Window.partitionBy(*partition_by).orderBy(order_by)
        
        if window_func == "sum":
            df_windowed = df.withColumn(
                f"{window_func}_{value_col}",
                sum(col(value_col)).over(window_spec)
            )
        elif window_func == "avg":
            df_windowed = df.withColumn(
                f"{window_func}_{value_col}",
                avg(col(value_col)).over(window_spec)
            )
        
        print(f"✅ Window function aplicada")
        return df_windowed
    
    def cache_dataframe(self, df: DataFrame, cache_type: str = "MEMORY"):
        """Cacheia DataFrame para reutilização"""
        if cache_type == "MEMORY":
            df.cache()
        elif cache_type == "DISK":
            df.persist(StorageLevel.DISK_ONLY)
        elif cache_type == "MEMORY_AND_DISK":
            df.persist(StorageLevel.MEMORY_AND_DISK)
        
        print(f"✅ DataFrame cacheado: {cache_type}")
        return df
```

## Passo 7: Otimizações de Performance

**src/utils/optimize.py:**
```python
from pyspark.sql import SparkSession, DataFrame

class SparkOptimizer:
    """Otimizações para Spark"""
    
    @staticmethod
    def repartition_by_column(df: DataFrame, column: str, num_partitions: int = None):
        """Reparticiona por coluna (melhora joins e agregações)"""
        if num_partitions:
            return df.repartition(num_partitions, column)
        else:
            return df.repartition(column)
    
    @staticmethod
    def coalesce_partitions(df: DataFrame, num_partitions: int):
        """Reduz número de partições (útil após filtros)"""
        return df.coalesce(num_partitions)
    
    @staticmethod
    def broadcast_join(df_small: DataFrame, df_large: DataFrame, join_key: str):
        """Faz broadcast join para tabelas pequenas"""
        from pyspark.sql.functions import broadcast
        
        return df_large.join(broadcast(df_small), on=join_key)
    
    @staticmethod
    def optimize_joins(df1: DataFrame, df2: DataFrame, join_key: str):
        """Otimiza joins"""
        # Se uma tabela é pequena (<100MB), usar broadcast
        # Caso contrário, garantir que ambas estão particionadas pela join key
        
        # Exemplo: verificar tamanho (simplificado)
        # Em produção, usar estimativas do Spark
        
        return df1.join(df2, on=join_key)
```

## Passo 8: Pipeline Completo

**main.py:**
```python
from src.utils.spark_config import create_spark_session
from src.extract.spark_extract import SparkExtractor
from src.transform.spark_transform import SparkTransformer
from src.load.spark_load import SparkLoader

def main():
    # Criar Spark Session
    spark = create_spark_session("Data Pipeline", memory="8g")
    
    print("=" * 60)
    print("PIPELINE COM PROCESSAMENTO DISTRIBUÍDO")
    print("=" * 60)
    
    # Extract
    extractor = SparkExtractor(spark)
    df_vendas = extractor.extract_csv("data/input/vendas.csv")
    df_produtos = extractor.extract_csv("data/input/produtos.csv")
    
    # Transform
    transformer = SparkTransformer(spark)
    
    # Limpar dados
    df_vendas_clean = transformer.clean_data(df_vendas)
    
    # Cache para reutilização
    df_vendas_clean.cache()
    
    # Join com produtos
    df_joined = transformer.join_dataframes(
        df_vendas_clean,
        df_produtos,
        "produto_id"
    )
    
    # Agregações
    df_agg = transformer.aggregate_data(
        df_joined,
        group_by_cols=["categoria", "mes"],
        agg_dict={
            "valor": "sum",
            "quantidade": "sum",
            "valor": "avg"
        }
    )
    
    # Load
    loader = SparkLoader(spark)
    loader.save_parquet(df_agg, "data/output/agregacoes")
    
    # Limpar cache
    df_vendas_clean.unpersist()
    
    print("\n✅ Pipeline concluído!")
    spark.stop()

if __name__ == "__main__":
    main()
```

## Passo 9: Monitoramento e Debugging

**src/utils/monitor.py:**
```python
from pyspark.sql import SparkSession

class SparkMonitor:
    """Monitora execução do Spark"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def get_stage_info(self):
        """Obtém informações dos stages"""
        status_tracker = self.spark.sparkContext.statusTracker()
        stage_infos = status_tracker.getActiveStageInfos()
        
        for stage_info in stage_infos:
            print(f"Stage {stage_info.stageId}: {stage_info.numTasks} tasks")
    
    def explain_plan(self, df):
        """Explica plano de execução"""
        df.explain(extended=True)
    
    def get_partition_info(self, df):
        """Obtém informações de partições"""
        rdd = df.rdd
        print(f"Número de partições: {rdd.getNumPartitions()}")
        print(f"Tamanho estimado: {rdd.count()} registros")
```

## Passo 10: Comparação Pandas vs Spark

**comparison.py:**
```python
import pandas as pd
from pyspark.sql import SparkSession

def pandas_approach(csv_path: str):
    """Abordagem com Pandas"""
    import time
    start = time.time()
    
    df = pd.read_csv(csv_path)
    result = df.groupby('categoria').agg({
        'valor': 'sum',
        'quantidade': 'sum'
    })
    
    elapsed = time.time() - start
    print(f"Pandas: {elapsed:.2f}s - {len(result)} grupos")
    return result

def spark_approach(csv_path: str):
    """Abordagem com Spark"""
    import time
    start = time.time()
    
    spark = SparkSession.builder.appName("Comparison").getOrCreate()
    df = spark.read.csv(csv_path, header=True, inferSchema=True)
    
    from pyspark.sql.functions import sum as spark_sum
    result = df.groupBy("categoria").agg(
        spark_sum("valor").alias("total_valor"),
        spark_sum("quantidade").alias("total_quantidade")
    )
    
    result_count = result.count()
    elapsed = time.time() - start
    
    print(f"Spark: {elapsed:.2f}s - {result_count} grupos")
    spark.stop()
    return result

# Testar com diferentes tamanhos de dados
# Para dados pequenos (<1GB): Pandas é mais rápido
# Para dados grandes (>10GB): Spark é necessário
```

## Checklist de Conclusão

- [ ] Spark configurado
- [ ] Extração distribuída implementada
- [ ] Transformações otimizadas
- [ ] Joins eficientes
- [ ] Cache implementado
- [ ] Particionamento otimizado
- [ ] Pipeline completo funcionando
- [ ] Performance monitorada
- [ ] Documentação completa
