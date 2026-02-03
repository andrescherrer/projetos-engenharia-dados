# Passo a Passo: Migração de Dados (OLTP → OLAP)

## Objetivo
Migrar dados de um sistema OLTP (transacional) para um sistema OLAP (analítico), implementando estratégias de extração eficientes e modelagem analítica.

## Pré-requisitos
- Python 3.8+
- PostgreSQL (OLTP) e PostgreSQL/BigQuery/Redshift (OLAP)
- Conhecimento de SQL e modelagem de dados

## Passo 1: Entender OLTP vs OLAP

**OLTP (Online Transaction Processing):**
- Otimizado para transações rápidas
- Dados normalizados (3NF)
- Muitas pequenas escritas
- Foco em consistência imediata

**OLAP (Online Analytical Processing):**
- Otimizado para consultas analíticas
- Dados desnormalizados (star schema)
- Poucas grandes leituras
- Foco em performance de queries

## Passo 2: Estrutura do Projeto

```
migracao-oltp-olap/
├── src/
│   ├── extract/
│   │   ├── full_extract.py
│   │   ├── incremental_extract.py
│   │   └── cdc_extract.py
│   ├── transform/
│   │   ├── dimensional_model.py
│   │   └── data_cleaning.py
│   ├── load/
│   │   └── olap_loader.py
│   └── validate/
│       └── migration_validator.py
├── sql/
│   ├── olap_schema.sql
│   └── migration_queries.sql
└── config/
    └── database_config.py
```

## Passo 3: Criar Schema OLAP (Star Schema)

**sql/olap_schema.sql:**
```sql
-- Schema analítico
CREATE SCHEMA IF NOT EXISTS analytics;

-- Tabela Fato (Fact Table)
CREATE TABLE analytics.fact_vendas (
    fact_id SERIAL PRIMARY KEY,
    date_id INT NOT NULL,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    store_id INT NOT NULL,
    quantidade INT NOT NULL,
    valor_total DECIMAL(10,2) NOT NULL,
    desconto DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Dimensão Data
CREATE TABLE analytics.dim_date (
    date_id SERIAL PRIMARY KEY,
    data DATE NOT NULL UNIQUE,
    ano INT NOT NULL,
    mes INT NOT NULL,
    dia INT NOT NULL,
    trimestre INT NOT NULL,
    dia_semana VARCHAR(20),
    is_fim_semana BOOLEAN
);

-- Dimensão Cliente
CREATE TABLE analytics.dim_customer (
    customer_id SERIAL PRIMARY KEY,
    customer_code VARCHAR(50) NOT NULL UNIQUE,
    nome VARCHAR(255),
    cidade VARCHAR(100),
    estado VARCHAR(50),
    segmento VARCHAR(50),
    valid_from DATE NOT NULL,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE
);

-- Dimensão Produto
CREATE TABLE analytics.dim_product (
    product_id SERIAL PRIMARY KEY,
    product_code VARCHAR(50) NOT NULL UNIQUE,
    nome VARCHAR(255),
    categoria VARCHAR(100),
    subcategoria VARCHAR(100),
    preco_base DECIMAL(10,2),
    valid_from DATE NOT NULL,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE
);

-- Dimensão Loja
CREATE TABLE analytics.dim_store (
    store_id SERIAL PRIMARY KEY,
    store_code VARCHAR(50) NOT NULL UNIQUE,
    nome VARCHAR(255),
    cidade VARCHAR(100),
    estado VARCHAR(50),
    regiao VARCHAR(50)
);

-- Índices para performance
CREATE INDEX idx_fact_vendas_date ON analytics.fact_vendas(date_id);
CREATE INDEX idx_fact_vendas_customer ON analytics.fact_vendas(customer_id);
CREATE INDEX idx_fact_vendas_product ON analytics.fact_vendas(product_id);
CREATE INDEX idx_fact_vendas_store ON analytics.fact_vendas(store_id);
```

## Passo 4: Implementar Extração Full Load

**src/extract/full_extract.py:**
```python
from sqlalchemy import create_engine
import pandas as pd
from datetime import datetime
from typing import Dict, List

class FullExtractor:
    """Extrai todos os dados de uma vez (usado na primeira migração)"""
    
    def __init__(self, source_connection: str):
        self.source_engine = create_engine(source_connection)
    
    def extract_table(self, table_name: str, schema: str = 'public') -> pd.DataFrame:
        """Extrai tabela completa"""
        query = f"SELECT * FROM {schema}.{table_name}"
        
        print(f"🔄 Extraindo tabela completa: {schema}.{table_name}")
        df = pd.read_sql(query, self.source_engine)
        print(f"✅ Extraídos {len(df)} registros")
        
        return df
    
    def extract_multiple_tables(self, tables: List[Dict]) -> Dict[str, pd.DataFrame]:
        """Extrai múltiplas tabelas"""
        results = {}
        
        for table_info in tables:
            table_name = table_info['name']
            schema = table_info.get('schema', 'public')
            df = self.extract_table(table_name, schema)
            results[table_name] = df
        
        return results

# Exemplo de uso
if __name__ == "__main__":
    extractor = FullExtractor("postgresql://user:pass@localhost/oltp_db")
    df_vendas = extractor.extract_table("vendas")
    print(df_vendas.head())
```

## Passo 5: Implementar Extração Incremental

**src/extract/incremental_extract.py:**
```python
from sqlalchemy import create_engine
import pandas as pd
from datetime import datetime, timedelta
from typing import Optional

class IncrementalExtractor:
    """Extrai apenas dados novos ou modificados"""
    
    def __init__(self, source_connection: str):
        self.source_engine = create_engine(source_connection)
    
    def get_last_extraction_time(self, target_connection: str, table_name: str) -> Optional[datetime]:
        """Obtém timestamp da última extração"""
        target_engine = create_engine(target_connection)
        
        query = f"""
            SELECT MAX(updated_at) as last_update 
            FROM analytics.{table_name}
        """
        
        try:
            result = pd.read_sql(query, target_engine)
            if not result.empty and result['last_update'].iloc[0] is not None:
                return pd.to_datetime(result['last_update'].iloc[0])
        except:
            pass
        
        return None
    
    def extract_incremental(
        self, 
        table_name: str, 
        timestamp_column: str = 'updated_at',
        target_connection: str = None
    ) -> pd.DataFrame:
        """Extrai dados incrementais"""
        
        # Obter última extração
        last_update = None
        if target_connection:
            last_update = self.get_last_extraction_time(target_connection, table_name)
        
        # Construir query incremental
        if last_update:
            query = f"""
                SELECT * FROM {table_name}
                WHERE {timestamp_column} > '{last_update}'
                ORDER BY {timestamp_column}
            """
            print(f"🔄 Extraindo dados desde {last_update}")
        else:
            # Primeira execução: extrair últimos 7 dias
            cutoff_date = datetime.now() - timedelta(days=7)
            query = f"""
                SELECT * FROM {table_name}
                WHERE {timestamp_column} > '{cutoff_date}'
                ORDER BY {timestamp_column}
            """
            print(f"🔄 Primeira extração: últimos 7 dias")
        
        df = pd.read_sql(query, self.source_engine)
        print(f"✅ Extraídos {len(df)} registros incrementais")
        
        return df
    
    def extract_by_date_range(
        self,
        table_name: str,
        start_date: datetime,
        end_date: datetime,
        date_column: str = 'created_at'
    ) -> pd.DataFrame:
        """Extrai dados de um range de datas específico"""
        query = f"""
            SELECT * FROM {table_name}
            WHERE {date_column} BETWEEN '{start_date}' AND '{end_date}'
            ORDER BY {date_column}
        """
        
        print(f"🔄 Extraindo dados de {start_date} a {end_date}")
        df = pd.read_sql(query, self.source_engine)
        print(f"✅ Extraídos {len(df)} registros")
        
        return df

# Exemplo de uso
if __name__ == "__main__":
    extractor = IncrementalExtractor("postgresql://user:pass@localhost/oltp_db")
    df_new = extractor.extract_incremental(
        "vendas",
        timestamp_column="updated_at",
        target_connection="postgresql://user:pass@localhost/olap_db"
    )
```

## Passo 6: Implementar Transformação para Modelo Dimensional

**src/transform/dimensional_model.py:**
```python
import pandas as pd
from datetime import datetime
from typing import Dict

class DimensionalTransformer:
    """Transforma dados normalizados em modelo dimensional"""
    
    def create_dim_date(self, start_date: datetime, end_date: datetime) -> pd.DataFrame:
        """Cria dimensão de data"""
        date_range = pd.date_range(start=start_date, end=end_date, freq='D')
        
        dim_date = pd.DataFrame({
            'data': date_range,
            'ano': date_range.year,
            'mes': date_range.month,
            'dia': date_range.day,
            'trimestre': date_range.quarter,
            'dia_semana': date_range.strftime('%A'),
            'is_fim_semana': date_range.weekday >= 5
        })
        
        dim_date['date_id'] = range(1, len(dim_date) + 1)
        
        return dim_date[['date_id', 'data', 'ano', 'mes', 'dia', 'trimestre', 'dia_semana', 'is_fim_semana']]
    
    def transform_to_fact_table(
        self,
        df_vendas: pd.DataFrame,
        dim_customer: pd.DataFrame,
        dim_product: pd.DataFrame,
        dim_store: pd.DataFrame,
        dim_date: pd.DataFrame
    ) -> pd.DataFrame:
        """Transforma dados transacionais em fato"""
        
        # Merge com dimensões
        fact = df_vendas.copy()
        
        # Merge com dim_date
        fact['data'] = pd.to_datetime(fact['data_venda'])
        fact = fact.merge(
            dim_date[['date_id', 'data']],
            on='data',
            how='left'
        )
        
        # Merge com dim_customer (usar versão atual)
        dim_customer_current = dim_customer[dim_customer['is_current'] == True]
        fact = fact.merge(
            dim_customer_current[['customer_id', 'customer_code']],
            left_on='cliente_id',
            right_on='customer_code',
            how='left'
        )
        
        # Merge com dim_product
        dim_product_current = dim_product[dim_product['is_current'] == True]
        fact = fact.merge(
            dim_product_current[['product_id', 'product_code']],
            left_on='produto_id',
            right_on='product_code',
            how='left'
        )
        
        # Merge com dim_store
        fact = fact.merge(
            dim_store[['store_id', 'store_code']],
            left_on='loja_id',
            right_on='store_code',
            how='left'
        )
        
        # Selecionar colunas da fato
        fact_table = fact[[
            'date_id',
            'customer_id',
            'product_id',
            'store_id',
            'quantidade',
            'valor_total',
            'desconto'
        ]].copy()
        
        return fact_table
    
    def create_scd_type2_dimension(
        self,
        df_source: pd.DataFrame,
        business_key: str,
        attributes: list
    ) -> pd.DataFrame:
        """Cria dimensão com SCD Type 2 (Slowly Changing Dimension)"""
        
        df_dim = df_source.copy()
        df_dim = df_dim.sort_values([business_key, 'updated_at'])
        
        # Criar versões
        df_dim['valid_from'] = df_dim['updated_at']
        df_dim['valid_to'] = None
        df_dim['is_current'] = False
        
        # Marcar versões atuais
        df_dim.loc[df_dim.groupby(business_key)['updated_at'].idxmax(), 'is_current'] = True
        
        # Preencher valid_to
        for key in df_dim[business_key].unique():
            mask = df_dim[business_key] == key
            versions = df_dim[mask].sort_values('valid_from')
            
            for i in range(len(versions) - 1):
                idx_current = versions.index[i]
                idx_next = versions.index[i + 1]
                df_dim.loc[idx_current, 'valid_to'] = df_dim.loc[idx_next, 'valid_from']
        
        return df_dim

# Exemplo de uso
if __name__ == "__main__":
    transformer = DimensionalTransformer()
    
    # Criar dimensão de data
    dim_date = transformer.create_dim_date(
        datetime(2020, 1, 1),
        datetime(2024, 12, 31)
    )
    
    print(f"Dimensão de data criada com {len(dim_date)} registros")
```

## Passo 7: Implementar Carga no OLAP

**src/load/olap_loader.py:**
```python
from sqlalchemy import create_engine
import pandas as pd
from typing import Dict

class OLAPLoader:
    """Carrega dados no Data Warehouse OLAP"""
    
    def __init__(self, target_connection: str):
        self.target_engine = create_engine(target_connection)
    
    def load_dimension(self, df: pd.DataFrame, table_name: str, schema: str = 'analytics', mode: str = 'append'):
        """Carrega dimensão"""
        print(f"🔄 Carregando dimensão {schema}.{table_name}")
        
        if_exists = 'append' if mode == 'append' else 'replace'
        
        df.to_sql(
            table_name,
            self.target_engine,
            schema=schema,
            if_exists=if_exists,
            index=False,
            method='multi',
            chunksize=1000
        )
        
        print(f"✅ Dimensão {table_name} carregada: {len(df)} registros")
    
    def load_fact(self, df: pd.DataFrame, table_name: str = 'fact_vendas', schema: str = 'analytics'):
        """Carrega fato"""
        print(f"🔄 Carregando fato {schema}.{table_name}")
        
        # Validar chaves estrangeiras antes de carregar
        self._validate_foreign_keys(df)
        
        df.to_sql(
            table_name,
            self.target_engine,
            schema=schema,
            if_exists='append',
            index=False,
            method='multi',
            chunksize=5000  # Chunks maiores para fatos
        )
        
        print(f"✅ Fato {table_name} carregado: {len(df)} registros")
    
    def _validate_foreign_keys(self, df: pd.DataFrame):
        """Valida chaves estrangeiras"""
        # Verificar se date_ids existem
        query = "SELECT DISTINCT date_id FROM analytics.dim_date"
        valid_dates = pd.read_sql(query, self.target_engine)['date_id'].tolist()
        
        invalid_dates = set(df['date_id'].unique()) - set(valid_dates)
        if invalid_dates:
            raise ValueError(f"Date IDs inválidos encontrados: {invalid_dates}")
        
        # Adicionar validações para outras FKs conforme necessário
        print("✅ Validação de chaves estrangeiras passou")
    
    def upsert_dimension(
        self,
        df: pd.DataFrame,
        table_name: str,
        business_key: str,
        schema: str = 'analytics'
    ):
        """Faz upsert em dimensão (insert ou update)"""
        # Implementar lógica de upsert
        # Para SCD Type 2, inserir novas versões e atualizar is_current
        pass

# Exemplo de uso
if __name__ == "__main__":
    loader = OLAPLoader("postgresql://user:pass@localhost/olap_db")
    # loader.load_dimension(dim_date, 'dim_date')
```

## Passo 8: Implementar Validação de Migração

**src/validate/migration_validator.py:**
```python
from sqlalchemy import create_engine
import pandas as pd

class MigrationValidator:
    """Valida integridade da migração"""
    
    def __init__(self, source_connection: str, target_connection: str):
        self.source_engine = create_engine(source_connection)
        self.target_engine = create_engine(target_connection)
    
    def validate_row_count(self, source_table: str, target_table: str, tolerance: float = 0.01) -> bool:
        """Valida contagem de linhas"""
        source_query = f"SELECT COUNT(*) as cnt FROM {source_table}"
        target_query = f"SELECT COUNT(*) as cnt FROM analytics.{target_table}"
        
        source_count = pd.read_sql(source_query, self.source_engine)['cnt'].iloc[0]
        target_count = pd.read_sql(target_query, self.target_engine)['cnt'].iloc[0]
        
        difference = abs(source_count - target_count)
        tolerance_count = source_count * tolerance
        
        print(f"Source: {source_count}, Target: {target_count}, Difference: {difference}")
        
        if difference > tolerance_count:
            print(f"❌ Diferença maior que tolerância ({tolerance_count})")
            return False
        
        print("✅ Validação de contagem passou")
        return True
    
    def validate_sum(self, source_table: str, source_column: str, 
                    target_table: str, target_column: str) -> bool:
        """Valida soma de valores"""
        source_query = f"SELECT SUM({source_column}) as total FROM {source_table}"
        target_query = f"SELECT SUM({target_column}) as total FROM analytics.{target_table}"
        
        source_sum = pd.read_sql(source_query, self.source_engine)['total'].iloc[0]
        target_sum = pd.read_sql(target_query, self.target_engine)['total'].iloc[0]
        
        difference = abs(source_sum - target_sum)
        
        print(f"Source sum: {source_sum}, Target sum: {target_sum}, Difference: {difference}")
        
        if difference > 0.01:  # Tolerância para arredondamento
            print("❌ Diferença na soma detectada")
            return False
        
        print("✅ Validação de soma passou")
        return True
    
    def validate_referential_integrity(self) -> bool:
        """Valida integridade referencial"""
        # Verificar se todas as FKs têm referências válidas
        query = """
            SELECT COUNT(*) as invalid_fks
            FROM analytics.fact_vendas f
            LEFT JOIN analytics.dim_date d ON f.date_id = d.date_id
            LEFT JOIN analytics.dim_customer c ON f.customer_id = c.customer_id
            LEFT JOIN analytics.dim_product p ON f.product_id = p.product_id
            LEFT JOIN analytics.dim_store s ON f.store_id = s.store_id
            WHERE d.date_id IS NULL 
               OR c.customer_id IS NULL 
               OR p.product_id IS NULL 
               OR s.store_id IS NULL
        """
        
        result = pd.read_sql(query, self.target_engine)
        invalid_count = result['invalid_fks'].iloc[0]
        
        if invalid_count > 0:
            print(f"❌ Encontradas {invalid_count} violações de integridade referencial")
            return False
        
        print("✅ Integridade referencial validada")
        return True

# Exemplo de uso
if __name__ == "__main__":
    validator = MigrationValidator(
        "postgresql://user:pass@localhost/oltp_db",
        "postgresql://user:pass@localhost/olap_db"
    )
    
    validator.validate_row_count("vendas", "fact_vendas")
    validator.validate_sum("vendas", "valor_total", "fact_vendas", "valor_total")
    validator.validate_referential_integrity()
```

## Passo 9: Pipeline Completo

**main.py:**
```python
from src.extract.full_extract import FullExtractor
from src.extract.incremental_extract import IncrementalExtractor
from src.transform.dimensional_model import DimensionalTransformer
from src.load.olap_loader import OLAPLoader
from src.validate.migration_validator import MigrationValidator
from datetime import datetime
import pandas as pd

def main():
    # Configurações
    oltp_conn = "postgresql://user:pass@localhost/oltp_db"
    olap_conn = "postgresql://user:pass@localhost/olap_db"
    
    # 1. Extração
    print("=" * 60)
    print("ETAPA 1: EXTRAÇÃO")
    print("=" * 60)
    
    extractor = FullExtractor(oltp_conn)
    df_vendas = extractor.extract_table("vendas")
    df_clientes = extractor.extract_table("clientes")
    df_produtos = extractor.extract_table("produtos")
    df_lojas = extractor.extract_table("lojas")
    
    # 2. Transformação
    print("\n" + "=" * 60)
    print("ETAPA 2: TRANSFORMAÇÃO")
    print("=" * 60)
    
    transformer = DimensionalTransformer()
    
    # Criar dimensões
    dim_date = transformer.create_dim_date(datetime(2020, 1, 1), datetime(2024, 12, 31))
    dim_customer = transformer.create_scd_type2_dimension(
        df_clientes,
        business_key='cliente_id',
        attributes=['nome', 'cidade', 'estado']
    )
    dim_product = transformer.create_scd_type2_dimension(
        df_produtos,
        business_key='produto_id',
        attributes=['nome', 'categoria', 'preco']
    )
    dim_store = df_lojas.copy()  # Dimensão simples
    
    # Criar fato
    fact_vendas = transformer.transform_to_fact_table(
        df_vendas,
        dim_customer,
        dim_product,
        dim_store,
        dim_date
    )
    
    # 3. Carga
    print("\n" + "=" * 60)
    print("ETAPA 3: CARGA")
    print("=" * 60)
    
    loader = OLAPLoader(olap_conn)
    loader.load_dimension(dim_date, 'dim_date', mode='replace')
    loader.load_dimension(dim_customer, 'dim_customer', mode='replace')
    loader.load_dimension(dim_product, 'dim_product', mode='replace')
    loader.load_dimension(dim_store, 'dim_store', mode='replace')
    loader.load_fact(fact_vendas)
    
    # 4. Validação
    print("\n" + "=" * 60)
    print("ETAPA 4: VALIDAÇÃO")
    print("=" * 60)
    
    validator = MigrationValidator(oltp_conn, olap_conn)
    validator.validate_row_count("vendas", "fact_vendas")
    validator.validate_sum("vendas", "valor_total", "fact_vendas", "valor_total")
    validator.validate_referential_integrity()
    
    print("\n✅ Migração concluída com sucesso!")

if __name__ == "__main__":
    main()
```

## Passo 10: Estratégias de Migração

### Full Load (Primeira Migração)
- Use quando: Primeira migração ou reconstrução completa
- Vantagens: Simples, garante consistência
- Desvantagens: Lento, pode impactar sistema fonte

### Incremental Load
- Use quando: Migrações periódicas (diárias, semanais)
- Vantagens: Rápido, baixo impacto
- Desvantagens: Requer controle de estado

### Change Data Capture (CDC)
- Use quando: Precisa de sincronização quase em tempo real
- Vantagens: Tempo real, eficiente
- Desvantagens: Mais complexo de implementar

## Checklist de Conclusão

- [ ] Schema OLAP criado (star schema)
- [ ] Extração full load implementada
- [ ] Extração incremental implementada
- [ ] Transformação para modelo dimensional
- [ ] SCD Type 2 implementado
- [ ] Carga no OLAP funcionando
- [ ] Validação de migração implementada
- [ ] Pipeline completo testado
- [ ] Documentação completa
