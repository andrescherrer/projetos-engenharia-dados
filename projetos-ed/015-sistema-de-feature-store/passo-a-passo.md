# Passo a Passo: Sistema de Feature Store

## Objetivo
Implementar um Feature Store para armazenar, versionar e servir features para modelos de ML, garantindo consistência entre treino e inferência.

## Pré-requisitos
- Python 3.8+
- Conhecimento de ML e features
- Redis (opcional, para feature serving)

## Passo 1: Escolher Ferramenta

**Opções:**
- **Feast:** Open-source, fácil de começar
- **Tecton:** Enterprise, muito completo
- **Hopsworks:** Open-source, completo

Vamos usar **Feast** (mais simples para começar).

## Passo 2: Instalar Feast

```bash
pip install feast[redis]
```

## Passo 3: Estrutura do Projeto

```
feature-store/
├── feature_repo/
│   ├── features/
│   │   ├── customer_features.py
│   │   └── product_features.py
│   ├── data_sources/
│   │   └── sources.py
│   └── feature_store.yaml
└── notebooks/
    └── feature_engineering.ipynb
```

## Passo 4: Configurar Feature Store

**feature_repo/feature_store.yaml:**
```yaml
project: customer_features
registry: data/registry.db
provider: local
online_store:
  type: redis
  connection_string: "localhost:6379"
```

## Passo 5: Definir Data Sources

**feature_repo/data_sources/sources.py:**
```python
from feast import FileSource, Entity
from datetime import timedelta

# Entidade Customer
customer_entity = Entity(
    name="customer",
    description="Customer entity",
    value_type="STRING"
)

# Data Source: Customer Transactions
customer_transactions_source = FileSource(
    name="customer_transactions",
    path="data/customer_transactions.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_at"
)

# Data Source: Customer Demographics
customer_demographics_source = FileSource(
    name="customer_demographics",
    path="data/customer_demographics.parquet",
    timestamp_field="event_timestamp"
)
```

## Passo 6: Definir Features

**feature_repo/features/customer_features.py:**
```python
from feast import Feature, FeatureView, ValueType
from feast.data_source import FileSource
from datetime import timedelta

# Features de transações
customer_transaction_features = FeatureView(
    name="customer_transaction_features",
    entities=["customer"],
    ttl=timedelta(days=90),
    features=[
        Feature(name="total_spent_30d", dtype=ValueType.FLOAT),
        Feature(name="transaction_count_30d", dtype=ValueType.INT64),
        Feature(name="avg_transaction_value_30d", dtype=ValueType.FLOAT),
        Feature(name="last_transaction_days_ago", dtype=ValueType.INT64),
    ],
    source=customer_transactions_source,
    tags={"team": "ml"}
)

# Features demográficas
customer_demographic_features = FeatureView(
    name="customer_demographic_features",
    entities=["customer"],
    ttl=timedelta(days=365),
    features=[
        Feature(name="age", dtype=ValueType.INT64),
        Feature(name="city", dtype=ValueType.STRING),
        Feature(name="signup_date", dtype=ValueType.UNIX_TIMESTAMP),
    ],
    source=customer_demographics_source,
    tags={"team": "ml"}
)
```

## Passo 7: Materializar Features

**scripts/materialize_features.py:**
```python
from feast import FeatureStore
from datetime import datetime, timedelta

def materialize_features():
    """Materializa features para online store"""
    fs = FeatureStore(repo_path="feature_repo")
    
    # Materializar últimas 30 dias
    end_date = datetime.now()
    start_date = end_date - timedelta(days=30)
    
    fs.materialize(
        start_date=start_date,
        end_date=end_date
    )
    
    print("✅ Features materializadas")

if __name__ == "__main__":
    materialize_features()
```

## Passo 8: Servir Features para Inferência

**src/serving/feature_serving.py:**
```python
from feast import FeatureStore
from typing import List, Dict

class FeatureServer:
    """Serve features para modelos ML"""
    
    def __init__(self, repo_path: str = "feature_repo"):
        self.fs = FeatureStore(repo_path=repo_path)
    
    def get_online_features(
        self,
        entity_ids: List[str],
        feature_names: List[str]
    ) -> Dict:
        """Obtém features do online store"""
        
        features = self.fs.get_online_features(
            features=feature_names,
            entity_rows=[{"customer": entity_id} for entity_id in entity_ids]
        )
        
        return features.to_dict()
    
    def get_historical_features(
        self,
        entity_df,
        feature_names: List[str]
    ):
        """Obtém features históricas para treino"""
        
        training_df = self.fs.get_historical_features(
            features=feature_names,
            entity_df=entity_df
        )
        
        return training_df.to_df()

# Exemplo de uso
server = FeatureServer()

# Para inferência (online)
online_features = server.get_online_features(
    entity_ids=["customer_123"],
    feature_names=[
        "customer_transaction_features:total_spent_30d",
        "customer_transaction_features:transaction_count_30d",
        "customer_demographic_features:age"
    ]
)

print(online_features)
```

## Passo 9: Integração com Pipeline de ML

**src/ml/training_pipeline.py:**
```python
import pandas as pd
from feast import FeatureStore
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

def train_model_with_features():
    """Treina modelo usando features do Feature Store"""
    
    fs = FeatureStore(repo_path="feature_repo")
    
    # Criar entity dataframe
    entity_df = pd.DataFrame({
        "customer": ["customer_1", "customer_2", "customer_3"],
        "event_timestamp": pd.to_datetime(["2024-01-01", "2024-01-02", "2024-01-03"])
    })
    
    # Obter features históricas
    training_df = fs.get_historical_features(
        features=[
            "customer_transaction_features:total_spent_30d",
            "customer_transaction_features:transaction_count_30d",
            "customer_demographic_features:age"
        ],
        entity_df=entity_df
    ).to_df()
    
    # Preparar dados para treino
    X = training_df.drop(columns=["customer", "event_timestamp"])
    y = training_df["target"]  # Assumindo que temos target
    
    # Split
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    # Treinar modelo
    model = RandomForestClassifier()
    model.fit(X_train, y_train)
    
    # Avaliar
    score = model.score(X_test, y_test)
    print(f"Model accuracy: {score}")
    
    return model

if __name__ == "__main__":
    model = train_model_with_features()
```

## Passo 10: Versionamento de Features

**src/versioning/feature_versioning.py:**
```python
from feast import FeatureStore
from datetime import datetime

class FeatureVersionManager:
    """Gerencia versionamento de features"""
    
    def __init__(self, repo_path: str = "feature_repo"):
        self.fs = FeatureStore(repo_path=repo_path)
    
    def create_feature_version(self, feature_name: str, version: str):
        """Cria nova versão de feature"""
        # Feast gerencia versionamento automaticamente
        # Versões são baseadas em timestamps
        pass
    
    def get_feature_version(self, feature_name: str, timestamp: datetime):
        """Obtém versão específica de feature"""
        # Obter features em um timestamp específico
        historical_features = self.fs.get_historical_features(
            features=[feature_name],
            entity_df=entity_df,
            feature_service=feature_service
        )
        
        return historical_features

# Exemplo
version_manager = FeatureVersionManager()

# Obter features em um timestamp específico
features_at_time = version_manager.get_feature_version(
    feature_name="customer_transaction_features:total_spent_30d",
    timestamp=datetime(2024, 1, 1)
)
```

## Checklist de Conclusão

- [ ] Feast instalado e configurado
- [ ] Data sources definidos
- [ ] Features definidas
- [ ] Materialização funcionando
- [ ] Feature serving implementado
- [ ] Integração com ML pipeline
- [ ] Versionamento configurado
- [ ] Documentação completa
