# Passo a Passo: Projeto de Streaming / Near Real-Time

## Objetivo
Implementar um pipeline de streaming para processar dados em tempo real usando Kafka e Spark Structured Streaming.

## Pré-requisitos
- Docker e Docker Compose
- Python 3.8+
- Conhecimento de Kafka e Spark

## Passo 1: Configurar Kafka

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: streaming_db
    ports:
      - "5432:5432"
```

## Passo 2: Criar Producer

**src/producer/kafka_producer.py:**
```python
from kafka import KafkaProducer
import json
import time
from datetime import datetime
from typing import Dict

class EventProducer:
    """Produz eventos para Kafka"""
    
    def __init__(self, bootstrap_servers: str = "localhost:9092"):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )
    
    def send_event(self, topic: str, event: Dict, key: str = None):
        """Envia evento para tópico"""
        future = self.producer.send(topic, value=event, key=key)
        
        try:
            record_metadata = future.get(timeout=10)
            print(f"✅ Evento enviado: topic={record_metadata.topic}, "
                  f"partition={record_metadata.partition}, "
                  f"offset={record_metadata.offset}")
        except Exception as e:
            print(f"❌ Erro ao enviar evento: {e}")
    
    def send_batch_events(self, topic: str, events: list):
        """Envia lote de eventos"""
        for event in events:
            key = event.get('user_id') or event.get('id')
            self.send_event(topic, event, key)
    
    def close(self):
        """Fecha producer"""
        self.producer.close()

# Exemplo: Gerar eventos de vendas
def generate_sales_events():
    producer = EventProducer()
    
    events = [
        {
            "event_id": f"evt_{i}",
            "user_id": f"user_{i % 100}",
            "product_id": f"prod_{i % 50}",
            "amount": 100.0 + (i * 10),
            "timestamp": datetime.now().isoformat(),
            "event_type": "purchase"
        }
        for i in range(1000)
    ]
    
    producer.send_batch_events("sales_events", events)
    producer.close()

if __name__ == "__main__":
    generate_sales_events()
```

## Passo 3: Spark Structured Streaming

**src/streaming/spark_streaming.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import (
    col, from_json, window, sum, count, avg,
    current_timestamp, to_timestamp, expr
)
from pyspark.sql.types import (
    StructType, StructField, StringType, DoubleType, TimestampType
)

class StreamingProcessor:
    """Processa streams com Spark Structured Streaming"""
    
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("StreamingPipeline") \
            .config("spark.sql.streaming.checkpointLocation", "/tmp/checkpoint") \
            .getOrCreate()
    
    def create_schema(self) -> StructType:
        """Define schema dos eventos"""
        return StructType([
            StructField("event_id", StringType(), True),
            StructField("user_id", StringType(), True),
            StructField("product_id", StringType(), True),
            StructField("amount", DoubleType(), True),
            StructField("timestamp", StringType(), True),
            StructField("event_type", StringType(), True)
        ])
    
    def read_from_kafka(self, topic: str, starting_offsets: str = "latest"):
        """Lê stream do Kafka"""
        df = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "localhost:9092") \
            .option("subscribe", topic) \
            .option("startingOffsets", starting_offsets) \
            .load()
        
        return df
    
    def parse_events(self, df, schema: StructType):
        """Parse eventos JSON"""
        # Converter valor de bytes para string JSON
        df_parsed = df.select(
            col("key").cast("string").alias("key"),
            from_json(col("value").cast("string"), schema).alias("data"),
            col("timestamp").alias("kafka_timestamp")
        )
        
        # Expandir colunas do JSON
        df_expanded = df_parsed.select(
            col("key"),
            col("data.*"),
            col("kafka_timestamp"),
            to_timestamp(col("data.timestamp")).alias("event_timestamp")
        )
        
        return df_expanded
    
    def aggregate_by_window(
        self,
        df,
        window_duration: str = "1 minute",
        slide_duration: str = "1 minute"
    ):
        """Agrega por janela de tempo"""
        df_agg = df \
            .withWatermark("event_timestamp", "10 seconds") \
            .groupBy(
                window(col("event_timestamp"), window_duration, slide_duration),
                col("product_id")
            ) \
            .agg(
                sum("amount").alias("total_amount"),
                count("*").alias("event_count"),
                avg("amount").alias("avg_amount")
            )
        
        return df_agg
    
    def write_to_console(self, df, output_mode: str = "append"):
        """Escreve para console (debug)"""
        query = df \
            .writeStream \
            .outputMode(output_mode) \
            .format("console") \
            .option("truncate", "false") \
            .start()
        
        return query
    
    def write_to_database(
        self,
        df,
        connection_string: str,
        table_name: str,
        output_mode: str = "update"
    ):
        """Escreve para banco de dados"""
        def write_batch(batch_df, batch_id):
            batch_df.write \
                .format("jdbc") \
                .option("url", connection_string) \
                .option("dbtable", table_name) \
                .option("user", "postgres") \
                .option("password", "postgres") \
                .mode("append") \
                .save()
        
        query = df \
            .writeStream \
            .outputMode(output_mode) \
            .foreachBatch(write_batch) \
            .start()
        
        return query
    
    def write_to_parquet(self, df, output_path: str, output_mode: str = "append"):
        """Escreve para Parquet"""
        query = df \
            .writeStream \
            .outputMode(output_mode) \
            .format("parquet") \
            .option("path", output_path) \
            .option("checkpointLocation", f"{output_path}/checkpoint") \
            .start()
        
        return query

# Pipeline completo
def run_streaming_pipeline():
    processor = StreamingProcessor()
    schema = processor.create_schema()
    
    # Ler do Kafka
    df_kafka = processor.read_from_kafka("sales_events")
    
    # Parse eventos
    df_events = processor.parse_events(df_kafka, schema)
    
    # Agregar por janela
    df_agg = processor.aggregate_by_window(
        df_events,
        window_duration="1 minute",
        slide_duration="1 minute"
    )
    
    # Escrever para console
    query = processor.write_to_console(df_agg)
    
    # Aguardar processamento
    query.awaitTermination()

if __name__ == "__main__":
    run_streaming_pipeline()
```

## Passo 4: Watermarks e Late Data

**src/streaming/watermarks.py:**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, window, max as spark_max

class WatermarkHandler:
    """Gerencia watermarks para dados atrasados"""
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def process_with_watermark(
        self,
        df,
        event_time_column: str = "event_timestamp",
        watermark_delay: str = "10 seconds"
    ):
        """Aplica watermark para lidar com dados atrasados"""
        df_with_watermark = df \
            .withWatermark(event_time_column, watermark_delay)
        
        return df_with_watermark
    
    def handle_late_data(self, df, late_data_threshold: str = "5 minutes"):
        """Processa dados atrasados separadamente"""
        # Dados dentro da janela
        df_on_time = df.filter(
            col("event_timestamp") >= 
            expr(f"current_timestamp() - interval {late_data_threshold}")
        )
        
        # Dados atrasados
        df_late = df.filter(
            col("event_timestamp") < 
            expr(f"current_timestamp() - interval {late_data_threshold}")
        )
        
        return df_on_time, df_late
```

## Passo 5: Janelas de Tempo

**src/streaming/windows.py:**
```python
from pyspark.sql.functions import window, col

class WindowProcessor:
    """Processa diferentes tipos de janelas"""
    
    @staticmethod
    def tumbling_window(df, window_duration: str, time_column: str = "timestamp"):
        """Janela deslizante sem sobreposição"""
        return df.groupBy(
            window(col(time_column), window_duration)
        )
    
    @staticmethod
    def sliding_window(
        df,
        window_duration: str,
        slide_duration: str,
        time_column: str = "timestamp"
    ):
        """Janela deslizante com sobreposição"""
        return df.groupBy(
            window(col(time_column), window_duration, slide_duration)
        )
    
    @staticmethod
    def session_window(
        df,
        session_gap: str,
        time_column: str = "timestamp"
    ):
        """Janela de sessão baseada em gap"""
        # Spark não tem session window nativo, usar workaround
        return df.groupBy(
            col("user_id"),
            window(col(time_column), session_gap)
        )
```

## Checklist de Conclusão

- [ ] Kafka configurado
- [ ] Producer implementado
- [ ] Spark Streaming configurado
- [ ] Pipeline de streaming funcionando
- [ ] Watermarks implementados
- [ ] Janelas de tempo configuradas
- [ ] Sink para banco de dados
- [ ] Tratamento de dados atrasados
- [ ] Monitoramento configurado
- [ ] Documentação completa
