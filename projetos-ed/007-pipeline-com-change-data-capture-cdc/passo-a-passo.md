# Passo a Passo: Pipeline com Change Data Capture (CDC)

## Objetivo
Implementar CDC para capturar mudanças em tempo real de um banco de dados transacional e replicá-las para um sistema analítico.

## Pré-requisitos
- Docker e Docker Compose
- Conhecimento de Kafka
- PostgreSQL com WAL habilitado
- Python 3.8+

## Passo 1: Entender CDC

**Change Data Capture (CDC):**
- Captura mudanças (INSERT, UPDATE, DELETE) em tempo real
- Usa logs de transação (WAL) do banco
- Não impacta performance do sistema fonte
- Replica apenas mudanças, não dados completos

**Ferramentas:**
- **Debezium:** Open-source, baseado em Kafka Connect
- **AWS DMS:** Serviço gerenciado da AWS
- **Fivetran:** SaaS, fácil de usar

Vamos usar **Debezium** (open-source e poderoso).

## Passo 2: Estrutura do Projeto

```
cdc-pipeline/
├── docker-compose.yml
├── debezium/
│   └── connector-config.json
├── src/
│   ├── consumer/
│   │   └── cdc_consumer.py
│   └── transform/
│       └── cdc_transformer.py
└── config/
    └── kafka_config.py
```

## Passo 3: Configurar Docker Compose

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

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
    image: debezium/postgres:13
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: source_db
    volumes:
      - postgres-data:/var/lib/postgresql/data

  postgres-target:
    image: postgres:13
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: target_db
    volumes:
      - postgres-target-data:/var/lib/postgresql/data

  connect:
    image: debezium/connect:latest
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - postgres
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses

volumes:
  postgres-data:
  postgres-target-data:
```

## Passo 4: Configurar PostgreSQL para CDC

**sql/setup_postgres.sql:**
```sql
-- Habilitar replicação lógica
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 4;
ALTER SYSTEM SET max_wal_senders = 4;

-- Criar tabela de exemplo
CREATE TABLE vendas (
    id SERIAL PRIMARY KEY,
    cliente_id INT NOT NULL,
    produto_id INT NOT NULL,
    valor DECIMAL(10,2) NOT NULL,
    quantidade INT NOT NULL,
    data_venda TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Criar função para atualizar updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Criar trigger
CREATE TRIGGER update_vendas_updated_at 
    BEFORE UPDATE ON vendas
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Inserir dados de teste
INSERT INTO vendas (cliente_id, produto_id, valor, quantidade) VALUES
(1, 101, 100.00, 2),
(2, 102, 250.50, 1),
(3, 101, 100.00, 3);
```

## Passo 5: Configurar Debezium Connector

**debezium/connector-config.json:**
```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname": "source_db",
    "database.server.name": "postgres-server",
    "table.include.list": "public.vendas",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_publication",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,source.ts_ms"
  }
}
```

## Passo 6: Registrar Connector

```bash
# Iniciar serviços
docker-compose up -d

# Aguardar serviços iniciarem
sleep 30

# Registrar connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @debezium/connector-config.json

# Verificar status
curl http://localhost:8083/connectors/postgres-connector/status
```

## Passo 7: Criar Consumer Python

**src/consumer/cdc_consumer.py:**
```python
from kafka import KafkaConsumer
import json
from sqlalchemy import create_engine
from sqlalchemy import text
from typing import Dict

class CDCConsumer:
    """Consome eventos CDC do Kafka e aplica no destino"""
    
    def __init__(self, kafka_broker: str, target_connection: str):
        self.consumer = KafkaConsumer(
            'postgres-server.public.vendas',
            bootstrap_servers=kafka_broker,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=True
        )
        self.target_engine = create_engine(target_connection)
        self._create_target_table()
    
    def _create_target_table(self):
        """Cria tabela de destino se não existir"""
        create_table_sql = """
        CREATE TABLE IF NOT EXISTS vendas_cdc (
            id INT PRIMARY KEY,
            cliente_id INT NOT NULL,
            produto_id INT NOT NULL,
            valor DECIMAL(10,2) NOT NULL,
            quantidade INT NOT NULL,
            data_venda TIMESTAMP,
            updated_at TIMESTAMP,
            cdc_op VARCHAR(10),
            cdc_timestamp TIMESTAMP,
            synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        """
        
        with self.target_engine.connect() as conn:
            conn.execute(text(create_table_sql))
            conn.commit()
    
    def process_cdc_event(self, event: Dict):
        """Processa um evento CDC"""
        op = event.get('op')  # c=create, u=update, d=delete
        after = event.get('after')
        before = event.get('before')
        
        if op == 'c':  # Create
            self._handle_insert(after)
        elif op == 'u':  # Update
            self._handle_update(after)
        elif op == 'd':  # Delete
            self._handle_delete(before)
        elif op == 'r':  # Read (snapshot)
            self._handle_insert(after)
    
    def _handle_insert(self, data: Dict):
        """Insere novo registro"""
        if not data:
            return
        
        insert_sql = """
        INSERT INTO vendas_cdc 
        (id, cliente_id, produto_id, valor, quantidade, data_venda, updated_at, cdc_op, cdc_timestamp)
        VALUES (:id, :cliente_id, :produto_id, :valor, :quantidade, :data_venda, :updated_at, 'INSERT', CURRENT_TIMESTAMP)
        ON CONFLICT (id) DO UPDATE SET
            cliente_id = EXCLUDED.cliente_id,
            produto_id = EXCLUDED.produto_id,
            valor = EXCLUDED.valor,
            quantidade = EXCLUDED.quantidade,
            data_venda = EXCLUDED.data_venda,
            updated_at = EXCLUDED.updated_at,
            cdc_op = 'INSERT',
            cdc_timestamp = CURRENT_TIMESTAMP
        """
        
        with self.target_engine.connect() as conn:
            conn.execute(text(insert_sql), data)
            conn.commit()
        
        print(f"✅ Inserido/Atualizado registro ID: {data.get('id')}")
    
    def _handle_update(self, data: Dict):
        """Atualiza registro existente"""
        if not data:
            return
        
        update_sql = """
        UPDATE vendas_cdc SET
            cliente_id = :cliente_id,
            produto_id = :produto_id,
            valor = :valor,
            quantidade = :quantidade,
            data_venda = :data_venda,
            updated_at = :updated_at,
            cdc_op = 'UPDATE',
            cdc_timestamp = CURRENT_TIMESTAMP
        WHERE id = :id
        """
        
        with self.target_engine.connect() as conn:
            conn.execute(text(update_sql), data)
            conn.commit()
        
        print(f"✅ Atualizado registro ID: {data.get('id')}")
    
    def _handle_delete(self, data: Dict):
        """Remove registro"""
        if not data:
            return
        
        delete_sql = "DELETE FROM vendas_cdc WHERE id = :id"
        
        with self.target_engine.connect() as conn:
            conn.execute(text(delete_sql), {'id': data.get('id')})
            conn.commit()
        
        print(f"✅ Deletado registro ID: {data.get('id')}")
    
    def start_consuming(self):
        """Inicia consumo de eventos"""
        print("🔄 Iniciando consumo de eventos CDC...")
        
        try:
            for message in self.consumer:
                event = message.value
                print(f"📨 Evento recebido: {event.get('op')} - ID: {event.get('after', event.get('before', {})).get('id')}")
                self.process_cdc_event(event)
        except KeyboardInterrupt:
            print("\n⏹️  Parando consumer...")
        finally:
            self.consumer.close()

# Exemplo de uso
if __name__ == "__main__":
    consumer = CDCConsumer(
        kafka_broker="localhost:9092",
        target_connection="postgresql://postgres:postgres@localhost:5433/target_db"
    )
    consumer.start_consuming()
```

## Passo 8: Testar CDC

```bash
# Em um terminal, iniciar consumer
python src/consumer/cdc_consumer.py

# Em outro terminal, fazer mudanças no banco fonte
psql -h localhost -U postgres -d source_db

# No psql:
INSERT INTO vendas (cliente_id, produto_id, valor, quantidade) VALUES (4, 103, 500.00, 1);
UPDATE vendas SET valor = 150.00 WHERE id = 1;
DELETE FROM vendas WHERE id = 2;

# Verificar no banco destino
psql -h localhost -p 5433 -U postgres -d target_db
SELECT * FROM vendas_cdc ORDER BY synced_at DESC;
```

## Passo 9: Transformações Avançadas

**src/transform/cdc_transformer.py:**
```python
from typing import Dict

class CDCTransformer:
    """Aplica transformações nos eventos CDC"""
    
    @staticmethod
    def enrich_event(event: Dict) -> Dict:
        """Enriquece evento com metadados"""
        event['processed_at'] = 'CURRENT_TIMESTAMP'
        event['source_system'] = 'postgres-source'
        return event
    
    @staticmethod
    def filter_event(event: Dict, filters: Dict) -> bool:
        """Filtra eventos baseado em critérios"""
        after = event.get('after', {})
        
        # Filtrar por valor mínimo
        if 'min_value' in filters:
            if after.get('valor', 0) < filters['min_value']:
                return False
        
        # Filtrar por categoria
        if 'allowed_categories' in filters:
            if after.get('categoria') not in filters['allowed_categories']:
                return False
        
        return True
    
    @staticmethod
    def transform_schema(event: Dict, schema_mapping: Dict) -> Dict:
        """Transforma schema do evento"""
        after = event.get('after', {})
        transformed = {}
        
        for old_key, new_key in schema_mapping.items():
            if old_key in after:
                transformed[new_key] = after[old_key]
        
        event['after'] = transformed
        return event
```

## Passo 10: Monitoramento

**src/monitor/cdc_monitor.py:**
```python
from kafka import KafkaConsumer
import json
from datetime import datetime
from collections import defaultdict

class CDCMonitor:
    """Monitora saúde do pipeline CDC"""
    
    def __init__(self, kafka_broker: str):
        self.consumer = KafkaConsumer(
            'postgres-server.public.vendas',
            bootstrap_servers=kafka_broker,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            consumer_timeout_ms=1000
        )
        self.stats = defaultdict(int)
    
    def collect_stats(self, duration_seconds: int = 60):
        """Coleta estatísticas por um período"""
        start_time = datetime.now()
        
        while (datetime.now() - start_time).seconds < duration_seconds:
            try:
                for message in self.consumer:
                    event = message.value
                    op = event.get('op')
                    self.stats[op] += 1
                    self.stats['total'] += 1
            except:
                break
        
        return dict(self.stats)
    
    def check_lag(self):
        """Verifica lag do consumer"""
        # Implementar verificação de lag
        pass

# Exemplo de uso
if __name__ == "__main__":
    monitor = CDCMonitor("localhost:9092")
    stats = monitor.collect_stats(60)
    print(f"Estatísticas: {stats}")
```

## Checklist de Conclusão

- [ ] Docker Compose configurado
- [ ] PostgreSQL com WAL habilitado
- [ ] Debezium connector configurado
- [ ] Consumer Python implementado
- [ ] Pipeline CDC testado
- [ ] Transformações implementadas
- [ ] Monitoramento configurado
- [ ] Documentação completa
