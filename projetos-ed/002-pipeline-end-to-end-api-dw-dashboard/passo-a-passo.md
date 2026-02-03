# Passo a Passo: Pipeline End-to-End (API → DW → Dashboard)

## Objetivo
Criar um pipeline completo que extrai dados de uma API pública, transforma, armazena em um Data Warehouse e cria visualizações em um dashboard.

## Pré-requisitos
- Python 3.8+
- PostgreSQL instalado (ou acesso a BigQuery/Redshift)
- Conta no GitHub (para versionamento)
- Conhecimento básico de SQL

## Passo 1: Escolher Fonte de Dados

**Opções de APIs públicas:**
- [OpenWeatherMap](https://openweathermap.org/api) - Dados meteorológicos
- [GitHub API](https://docs.github.com/en/rest) - Dados de repositórios
- [Spotify API](https://developer.spotify.com/documentation/web-api) - Dados de músicas
- [Dados Abertos do Brasil](https://dados.gov.br/) - Dados governamentais

**Para este tutorial, usaremos dados públicos do IBGE:**
- API de população: `https://servicodados.ibge.gov.br/api/v1/pesquisas`

## Passo 2: Configurar Ambiente

```bash
# Criar projeto
mkdir pipeline-end-to-end
cd pipeline-end-to-end

# Criar ambiente virtual
python -m venv venv
source venv/bin/activate  # Linux/Mac

# Instalar dependências
pip install requests pandas sqlalchemy psycopg2-binary prefect matplotlib seaborn
pip freeze > requirements.txt
```

## Passo 3: Estrutura do Projeto

```
pipeline-end-to-end/
├── src/
│   ├── extract/
│   │   └── api_extractor.py
│   ├── transform/
│   │   └── transformer.py
│   ├── load/
│   │   └── dw_loader.py
│   └── utils/
│       └── database.py
├── dags/
│   └── pipeline_dag.py
├── sql/
│   └── create_tables.sql
├── dashboard/
│   └── generate_dashboard.py
├── config/
│   └── config.yaml
├── .env.example
└── README.md
```

## Passo 4: Configurar Banco de Dados

**sql/create_tables.sql:**
```sql
-- Criar schema
CREATE SCHEMA IF NOT EXISTS analytics;

-- Tabela de dados brutos (staging)
CREATE TABLE IF NOT EXISTS analytics.staging_api_data (
    id SERIAL PRIMARY KEY,
    data_raw JSONB,
    source VARCHAR(255),
    extracted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de dados transformados
CREATE TABLE IF NOT EXISTS analytics.transformed_data (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(255),
    valor NUMERIC,
    categoria VARCHAR(100),
    data_referencia DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para performance
CREATE INDEX idx_transformed_data_categoria ON analytics.transformed_data(categoria);
CREATE INDEX idx_transformed_data_data_referencia ON analytics.transformed_data(data_referencia);
```

## Passo 5: Implementar Extração de API

**src/extract/api_extractor.py:**
```python
import requests
import json
from datetime import datetime
from typing import Dict, List

class APIExtractor:
    def __init__(self, base_url: str, api_key: str = None):
        self.base_url = base_url
        self.headers = {}
        if api_key:
            self.headers['Authorization'] = f'Bearer {api_key}'
    
    def fetch_data(self, endpoint: str, params: Dict = None) -> List[Dict]:
        """Extrai dados de uma API"""
        url = f"{self.base_url}/{endpoint}"
        
        try:
            response = requests.get(url, headers=self.headers, params=params)
            response.raise_for_status()
            data = response.json()
            print(f"✅ Extraídos {len(data)} registros de {endpoint}")
            return data
        except requests.exceptions.RequestException as e:
            print(f"❌ Erro ao extrair dados: {e}")
            raise
    
    def extract_with_pagination(self, endpoint: str, page_size: int = 100) -> List[Dict]:
        """Extrai dados com paginação"""
        all_data = []
        page = 1
        
        while True:
            params = {'page': page, 'per_page': page_size}
            data = self.fetch_data(endpoint, params)
            
            if not data:
                break
            
            all_data.extend(data)
            page += 1
            
            # Limite de segurança
            if page > 100:
                break
        
        return all_data

# Exemplo de uso
if __name__ == "__main__":
    # Exemplo com API pública do IBGE
    extractor = APIExtractor("https://servicodados.ibge.gov.br/api/v1")
    dados = extractor.fetch_data("localidades/estados")
    print(f"Total de estados: {len(dados)}")
```

## Passo 6: Implementar Transformação

**src/transform/transformer.py:**
```python
import pandas as pd
from typing import List, Dict
from datetime import datetime

class DataTransformer:
    def __init__(self):
        pass
    
    def normalize_data(self, raw_data: List[Dict]) -> pd.DataFrame:
        """Normaliza dados brutos em DataFrame"""
        df = pd.DataFrame(raw_data)
        return df
    
    def clean_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """Limpa e padroniza dados"""
        # Remove duplicatas
        df = df.drop_duplicates()
        
        # Padroniza nomes de colunas
        df.columns = df.columns.str.lower().str.replace(' ', '_')
        
        # Remove valores nulos críticos
        df = df.dropna(subset=['id'])  # Ajuste conforme sua estrutura
        
        return df
    
    def enrich_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """Adiciona colunas calculadas"""
        df['processed_at'] = datetime.now()
        df['year'] = pd.to_datetime(df.get('data_referencia', datetime.now())).dt.year
        
        return df
    
    def transform(self, raw_data: List[Dict]) -> pd.DataFrame:
        """Pipeline completo de transformação"""
        df = self.normalize_data(raw_data)
        df = self.clean_data(df)
        df = self.enrich_data(df)
        
        print(f"✅ Transformados {len(df)} registros")
        return df

# Exemplo de uso
if __name__ == "__main__":
    transformer = DataTransformer()
    # raw_data seria o resultado da extração
    # df_transformed = transformer.transform(raw_data)
```

## Passo 7: Implementar Carga no Data Warehouse

**src/load/dw_loader.py:**
```python
import pandas as pd
from sqlalchemy import create_engine
from src.utils.database import get_connection_string
from typing import List, Dict

class DWLoader:
    def __init__(self, connection_string: str):
        self.engine = create_engine(connection_string)
    
    def load_staging(self, raw_data: List[Dict], source: str):
        """Carrega dados brutos na camada staging"""
        import json
        from datetime import datetime
        
        staging_data = {
            'data_raw': [json.dumps(d) for d in raw_data],
            'source': source,
            'extracted_at': datetime.now()
        }
        
        df_staging = pd.DataFrame(staging_data)
        df_staging.to_sql(
            'staging_api_data',
            self.engine,
            schema='analytics',
            if_exists='append',
            index=False
        )
        print(f"✅ Carregados {len(df_staging)} registros em staging")
    
    def load_transformed(self, df: pd.DataFrame, table_name: str = 'transformed_data'):
        """Carrega dados transformados"""
        df.to_sql(
            table_name,
            self.engine,
            schema='analytics',
            if_exists='append',
            index=False
        )
        print(f"✅ Carregados {len(df)} registros transformados")
    
    def upsert_data(self, df: pd.DataFrame, table_name: str, unique_key: str):
        """Faz upsert (insert ou update) de dados"""
        # Implementação de upsert usando SQL
        for _, row in df.iterrows():
            # Verificar se existe
            query = f"""
                SELECT COUNT(*) FROM analytics.{table_name}
                WHERE {unique_key} = %s
            """
            # Implementar lógica de upsert
            pass

# Exemplo de uso
if __name__ == "__main__":
    from dotenv import load_dotenv
    import os
    
    load_dotenv()
    conn_str = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    
    loader = DWLoader(conn_str)
    # loader.load_transformed(df_transformed)
```

## Passo 8: Criar Pipeline com Prefect

**dags/pipeline_dag.py:**
```python
from prefect import flow, task
from src.extract.api_extractor import APIExtractor
from src.transform.transformer import DataTransformer
from src.load.dw_loader import DWLoader
import os
from dotenv import load_dotenv

load_dotenv()

@task
def extract_task():
    """Task de extração"""
    extractor = APIExtractor("https://servicodados.ibge.gov.br/api/v1")
    data = extractor.fetch_data("localidades/estados")
    return data

@task
def transform_task(raw_data):
    """Task de transformação"""
    transformer = DataTransformer()
    df_transformed = transformer.transform(raw_data)
    return df_transformed

@task
def load_task(df_transformed):
    """Task de carga"""
    conn_str = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    loader = DWLoader(conn_str)
    loader.load_transformed(df_transformed)
    return "Success"

@flow(name="Pipeline End-to-End")
def pipeline_flow():
    """Flow principal do pipeline"""
    print("🔄 Iniciando pipeline...")
    
    # Extract
    raw_data = extract_task()
    
    # Transform
    df_transformed = transform_task(raw_data)
    
    # Load
    result = load_task(df_transformed)
    
    print("✅ Pipeline concluído!")
    return result

if __name__ == "__main__":
    pipeline_flow()
```

## Passo 9: Criar Dashboard

**dashboard/generate_dashboard.py:**
```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sqlalchemy import create_engine
import os
from dotenv import load_dotenv

load_dotenv()

def generate_dashboard():
    """Gera dashboard com visualizações"""
    conn_str = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    engine = create_engine(conn_str)
    
    # Query dados transformados
    query = "SELECT * FROM analytics.transformed_data"
    df = pd.read_sql(query, engine)
    
    # Configurar estilo
    sns.set_style("whitegrid")
    fig, axes = plt.subplots(2, 2, figsize=(15, 10))
    
    # Gráfico 1: Distribuição por categoria
    df['categoria'].value_counts().plot(kind='bar', ax=axes[0, 0])
    axes[0, 0].set_title('Distribuição por Categoria')
    axes[0, 0].set_xlabel('Categoria')
    axes[0, 0].set_ylabel('Quantidade')
    
    # Gráfico 2: Valores ao longo do tempo
    df.groupby('data_referencia')['valor'].sum().plot(ax=axes[0, 1])
    axes[0, 1].set_title('Evolução dos Valores')
    axes[0, 1].set_xlabel('Data')
    axes[0, 1].set_ylabel('Valor Total')
    
    # Gráfico 3: Boxplot de valores
    df.boxplot(column='valor', by='categoria', ax=axes[1, 0])
    axes[1, 0].set_title('Distribuição de Valores por Categoria')
    
    # Gráfico 4: Tabela de resumo
    summary = df.groupby('categoria').agg({
        'valor': ['sum', 'mean', 'count']
    }).round(2)
    axes[1, 1].axis('tight')
    axes[1, 1].axis('off')
    axes[1, 1].table(cellText=summary.values,
                     rowLabels=summary.index,
                     colLabels=summary.columns.get_level_values(1),
                     cellLoc='center',
                     loc='center')
    axes[1, 1].set_title('Resumo Estatístico')
    
    plt.tight_layout()
    plt.savefig('dashboard/dashboard.png', dpi=300, bbox_inches='tight')
    print("✅ Dashboard gerado em dashboard/dashboard.png")

if __name__ == "__main__":
    generate_dashboard()
```

## Passo 10: Configurar Variáveis de Ambiente

**.env.example:**
```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=analytics_db
DB_USER=postgres
DB_PASSWORD=your_password
API_KEY=your_api_key_if_needed
```

## Passo 11: Executar Pipeline

```bash
# Executar pipeline
python -m dags.pipeline_dag

# Ou com Prefect UI
prefect server start
# Acessar http://localhost:4200
```

## Passo 12: Próximos Passos

1. **Adicionar agendamento:** Configure o Prefect para rodar diariamente
2. **Adicionar testes:** Crie testes unitários para cada componente
3. **Melhorar dashboard:** Use Streamlit ou Dash para dashboard interativo
4. **Adicionar alertas:** Configure alertas para falhas no pipeline
5. **Versionamento:** Use Git para versionar o código

## Recursos de Aprendizado

- [Prefect Documentation](https://docs.prefect.io/)
- [SQLAlchemy Tutorial](https://docs.sqlalchemy.org/en/14/tutorial/)
- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [Matplotlib Gallery](https://matplotlib.org/stable/gallery/)

## Checklist de Conclusão

- [ ] Ambiente configurado
- [ ] Banco de dados criado
- [ ] Extração de API implementada
- [ ] Transformação implementada
- [ ] Carga no DW implementada
- [ ] Pipeline orquestrado funcionando
- [ ] Dashboard gerado
- [ ] Código versionado no Git
- [ ] Documentação completa
