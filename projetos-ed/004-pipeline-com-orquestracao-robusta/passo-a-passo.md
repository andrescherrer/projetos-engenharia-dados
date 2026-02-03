# Passo a Passo: Pipeline com Orquestração Robusta

## Objetivo
Criar um pipeline robusto com orquestração profissional usando Airflow ou Prefect, incluindo retries, SLAs, cache e tratamento de falhas.

## Pré-requisitos
- Python 3.8+
- Docker (para Airflow) ou Prefect instalado
- Conhecimento básico de pipelines de dados

## Escolha: Airflow vs Prefect

**Airflow:** Mais maduro, amplamente usado, mas mais complexo de configurar
**Prefect:** Mais moderno, Python-native, mais fácil de começar

Vamos usar **Prefect** para este tutorial (mais simples), mas os conceitos se aplicam a ambos.

## Passo 1: Configurar Ambiente

```bash
# Instalar Prefect
pip install prefect prefect-sqlalchemy prefect-docker

# Ou instalar Airflow
pip install apache-airflow
```

## Passo 2: Estrutura do Projeto

```
pipeline-orquestrado/
├── flows/
│   ├── extract_flow.py
│   ├── transform_flow.py
│   └── main_flow.py
├── tasks/
│   ├── extract_tasks.py
│   ├── transform_tasks.py
│   └── load_tasks.py
├── config/
│   └── settings.py
├── utils/
│   └── logging.py
└── tests/
    └── test_flows.py
```

## Passo 3: Configurar Prefect

**config/settings.py:**
```python
from prefect import settings

# Configurações do Prefect
PREFECT_API_URL = "http://localhost:4200/api"
PREFECT_LOGGING_LEVEL = "INFO"

# Configurações do pipeline
RETRY_ATTEMPTS = 3
RETRY_DELAY_SECONDS = 60
TIMEOUT_SECONDS = 3600
```

## Passo 4: Criar Tasks com Retry e Timeout

**tasks/extract_tasks.py:**
```python
from prefect import task
from prefect.tasks import task_input_hash
from datetime import timedelta
import requests
import pandas as pd

@task(
    retries=3,
    retry_delay_seconds=60,
    timeout_seconds=300,
    cache_key_fn=task_input_hash,
    cache_expiration=timedelta(hours=1),
    log_prints=True
)
def extract_from_api(url: str, params: dict = None) -> pd.DataFrame:
    """Extrai dados de uma API com retry automático"""
    print(f"🔄 Extraindo dados de {url}")
    
    try:
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()
        
        df = pd.DataFrame(data)
        print(f"✅ Extraídos {len(df)} registros")
        return df
    
    except requests.exceptions.RequestException as e:
        print(f"❌ Erro ao extrair dados: {e}")
        raise  # Prefect vai fazer retry automaticamente

@task(
    retries=2,
    retry_delay_seconds=30
)
def extract_from_database(connection_string: str, query: str) -> pd.DataFrame:
    """Extrai dados de banco de dados"""
    from sqlalchemy import create_engine
    
    print(f"🔄 Executando query no banco de dados")
    engine = create_engine(connection_string)
    
    try:
        df = pd.read_sql(query, engine)
        print(f"✅ Extraídos {len(df)} registros do banco")
        return df
    except Exception as e:
        print(f"❌ Erro ao extrair do banco: {e}")
        raise
```

## Passo 5: Criar Tasks de Transformação

**tasks/transform_tasks.py:**
```python
from prefect import task
import pandas as pd

@task(
    retries=2,
    retry_delay_seconds=30,
    log_prints=True
)
def clean_data(df: pd.DataFrame) -> pd.DataFrame:
    """Limpa dados"""
    print(f"🔄 Limpando {len(df)} registros")
    
    # Remove duplicatas
    df_clean = df.drop_duplicates()
    
    # Remove nulos em colunas críticas
    if 'id' in df_clean.columns:
        df_clean = df_clean.dropna(subset=['id'])
    
    print(f"✅ Dados limpos: {len(df_clean)} registros")
    return df_clean

@task(
    retries=2,
    log_prints=True
)
def transform_data(df: pd.DataFrame, transformations: dict) -> pd.DataFrame:
    """Aplica transformações"""
    print(f"🔄 Aplicando transformações")
    
    df_transformed = df.copy()
    
    # Renomear colunas
    if 'rename_columns' in transformations:
        df_transformed = df_transformed.rename(columns=transformations['rename_columns'])
    
    # Adicionar colunas calculadas
    if 'calculated_columns' in transformations:
        for col_name, expression in transformations['calculated_columns'].items():
            df_transformed[col_name] = eval(expression, {'df': df_transformed})
    
    print(f"✅ Transformações aplicadas")
    return df_transformed

@task(
    retries=1,
    log_prints=True
)
def validate_data(df: pd.DataFrame, validation_rules: dict) -> bool:
    """Valida dados"""
    print(f"🔄 Validando dados")
    
    # Validação de linhas mínimas
    if 'min_rows' in validation_rules:
        if len(df) < validation_rules['min_rows']:
            raise ValueError(f"Dados têm menos de {validation_rules['min_rows']} linhas")
    
    # Validação de colunas obrigatórias
    if 'required_columns' in validation_rules:
        missing = set(validation_rules['required_columns']) - set(df.columns)
        if missing:
            raise ValueError(f"Colunas faltando: {missing}")
    
    print(f"✅ Validação passou")
    return True
```

## Passo 6: Criar Tasks de Carga

**tasks/load_tasks.py:**
```python
from prefect import task
from sqlalchemy import create_engine
import pandas as pd

@task(
    retries=3,
    retry_delay_seconds=60,
    log_prints=True
)
def load_to_database(df: pd.DataFrame, connection_string: str, table_name: str, schema: str = 'public'):
    """Carrega dados no banco"""
    print(f"🔄 Carregando {len(df)} registros em {schema}.{table_name}")
    
    engine = create_engine(connection_string)
    
    try:
        df.to_sql(
            table_name,
            engine,
            schema=schema,
            if_exists='append',
            index=False,
            method='multi',
            chunksize=1000
        )
        print(f"✅ Dados carregados com sucesso")
    except Exception as e:
        print(f"❌ Erro ao carregar dados: {e}")
        raise

@task(
    retries=2,
    log_prints=True
)
def send_notification(message: str, channel: str = 'email'):
    """Envia notificação"""
    print(f"📧 Enviando notificação: {message}")
    # Implementar lógica de notificação
    # Pode ser email, Slack, etc.
    pass
```

## Passo 7: Criar Flow Principal

**flows/main_flow.py:**
```python
from prefect import flow, get_run_logger
from prefect.task_runners import ConcurrentTaskRunner
from datetime import datetime, timedelta
from tasks.extract_tasks import extract_from_api, extract_from_database
from tasks.transform_tasks import clean_data, transform_data, validate_data
from tasks.load_tasks import load_to_database, send_notification
import pandas as pd

@flow(
    name="Pipeline Principal",
    task_runner=ConcurrentTaskRunner(),
    log_prints=True
)
def main_pipeline_flow(
    api_url: str,
    db_connection: str,
    target_table: str
):
    """Flow principal do pipeline"""
    logger = get_run_logger()
    logger.info("🚀 Iniciando pipeline principal")
    
    try:
        # Extract - pode rodar em paralelo
        df_api = extract_from_api(api_url)
        df_db = extract_from_database(db_connection, "SELECT * FROM source_table")
        
        # Combinar dados
        df_combined = pd.concat([df_api, df_db], ignore_index=True)
        
        # Transform
        df_cleaned = clean_data(df_combined)
        df_transformed = transform_data(
            df_cleaned,
            {
                'rename_columns': {'old_name': 'new_name'},
                'calculated_columns': {'total': 'df["valor"] * df["quantidade"]'}
            }
        )
        
        # Validate
        validate_data(df_transformed, {
            'min_rows': 100,
            'required_columns': ['id', 'nome', 'valor']
        })
        
        # Load
        load_to_database(df_transformed, db_connection, target_table)
        
        # Notificação de sucesso
        send_notification("Pipeline executado com sucesso!")
        
        logger.info("✅ Pipeline concluído com sucesso")
        return "Success"
    
    except Exception as e:
        logger.error(f"❌ Erro no pipeline: {e}")
        send_notification(f"Pipeline falhou: {str(e)}")
        raise

@flow(
    name="Pipeline com SLA",
    log_prints=True
)
def pipeline_with_sla():
    """Pipeline com SLA definido"""
    from prefect import get_client
    
    # Definir SLA de 1 hora
    sla = datetime.now() + timedelta(hours=1)
    
    try:
        result = main_pipeline_flow(
            api_url="https://api.example.com/data",
            db_connection="postgresql://user:pass@localhost/db",
            target_table="target_table"
        )
        
        # Verificar se completou dentro do SLA
        if datetime.now() > sla:
            send_notification("⚠️ Pipeline completou mas ultrapassou SLA")
        
        return result
    
    except Exception as e:
        send_notification(f"❌ Pipeline falhou e não completou dentro do SLA: {e}")
        raise
```

## Passo 8: Configurar Agendamento

**flows/scheduled_flow.py:**
```python
from prefect import flow
from prefect.schedules import CronSchedule
from flows.main_flow import main_pipeline_flow

# Agendar para rodar diariamente às 2h da manhã
@flow(
    name="Pipeline Agendado",
    schedule=CronSchedule(cron="0 2 * * *", timezone="America/Sao_Paulo")
)
def scheduled_pipeline():
    """Pipeline agendado para rodar diariamente"""
    return main_pipeline_flow(
        api_url="https://api.example.com/data",
        db_connection="postgresql://user:pass@localhost/db",
        target_table="target_table"
    )
```

## Passo 9: Configurar Observabilidade

**utils/logging.py:**
```python
import logging
from prefect import get_run_logger

def setup_logging():
    """Configura logging estruturado"""
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler('pipeline.log'),
            logging.StreamHandler()
        ]
    )

def log_metrics(metrics: dict):
    """Registra métricas"""
    logger = get_run_logger()
    for key, value in metrics.items():
        logger.info(f"METRIC: {key}={value}")
```

## Passo 10: Criar Deployment

**deploy.py:**
```python
from prefect.deployments import Deployment
from prefect.server.schemas.schedules import CronSchedule
from flows.main_flow import main_pipeline_flow

deployment = Deployment.build_from_flow(
    flow=main_pipeline_flow,
    name="production-deployment",
    schedule=CronSchedule(cron="0 2 * * *"),
    work_queue_name="production",
    parameters={
        "api_url": "https://api.example.com/data",
        "db_connection": "postgresql://user:pass@localhost/db",
        "target_table": "target_table"
    }
)

if __name__ == "__main__":
    deployment.apply()
```

## Passo 11: Executar Prefect Server

```bash
# Iniciar servidor Prefect
prefect server start

# Em outro terminal, executar o flow
python flows/main_flow.py

# Ou criar deployment
python deploy.py
```

## Passo 12: Monitoramento e Alertas

**utils/alerts.py:**
```python
from prefect import get_run_logger
import smtplib
from email.mime.text import MIMEText

def send_alert(subject: str, message: str, recipients: list):
    """Envia alerta por email"""
    logger = get_run_logger()
    
    try:
        msg = MIMEText(message)
        msg['Subject'] = subject
        msg['From'] = "pipeline@company.com"
        msg['To'] = ", ".join(recipients)
        
        # Configurar servidor SMTP
        server = smtplib.SMTP('smtp.company.com', 587)
        server.starttls()
        server.login('user', 'password')
        server.send_message(msg)
        server.quit()
        
        logger.info(f"✅ Alerta enviado para {recipients}")
    except Exception as e:
        logger.error(f"❌ Erro ao enviar alerta: {e}")
```

## Passo 13: Boas Práticas

1. **Idempotência:** Tasks devem poder rodar múltiplas vezes sem efeitos colaterais
2. **Cache:** Use cache para evitar reprocessamento desnecessário
3. **Retries:** Configure retries apropriados para falhas temporárias
4. **Timeouts:** Defina timeouts para evitar tasks travadas
5. **Logging:** Use logging estruturado para debugging
6. **Validação:** Valide dados antes de carregar
7. **Notificações:** Configure alertas para falhas críticas
8. **Ambientes:** Separe dev/staging/prod

## Checklist de Conclusão

- [ ] Ambiente Prefect/Airflow configurado
- [ ] Tasks com retry implementadas
- [ ] Tasks com timeout configurado
- [ ] Cache implementado onde apropriado
- [ ] Flow principal criado
- [ ] Agendamento configurado
- [ ] SLA definido
- [ ] Logging estruturado
- [ ] Alertas configurados
- [ ] Deployment criado
- [ ] Testes escritos
- [ ] Documentação completa
