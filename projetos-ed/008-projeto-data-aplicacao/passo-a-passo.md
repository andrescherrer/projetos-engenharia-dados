# Passo a Passo: Projeto "Data + Aplicação"

## Objetivo
Criar uma aplicação que expõe dados analíticos através de uma API REST, demonstrando como dados podem ser consumidos por aplicações.

## Pré-requisitos
- Python 3.8+
- FastAPI ou Flask
- PostgreSQL ou outro banco de dados
- Conhecimento básico de APIs REST

## Passo 1: Estrutura do Projeto

```
data-app/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models/
│   │   └── data_models.py
│   ├── routes/
│   │   └── analytics.py
│   ├── services/
│   │   └── data_service.py
│   └── auth/
│       └── security.py
├── database/
│   └── queries.sql
└── requirements.txt
```

## Passo 2: Configurar Dependências

**requirements.txt:**
```
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
pydantic==2.5.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
pandas==2.1.3
```

## Passo 3: Criar Modelos de Dados

**app/models/data_models.py:**
```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class SalesSummary(BaseModel):
    date: datetime
    total_sales: float
    quantity: int
    avg_ticket: float

class CustomerMetrics(BaseModel):
    customer_id: int
    total_spent: float
    total_orders: int
    avg_order_value: float
    last_order_date: datetime

class ProductPerformance(BaseModel):
    product_id: int
    product_name: str
    total_revenue: float
    units_sold: int
    avg_price: float

class TimeSeriesData(BaseModel):
    period: str
    value: float
    metric: str
```

## Passo 4: Criar Serviço de Dados

**app/services/data_service.py:**
```python
from sqlalchemy import create_engine, text
import pandas as pd
from typing import List, Dict
from datetime import datetime, timedelta
from app.models.data_models import SalesSummary, CustomerMetrics, ProductPerformance

class DataService:
    """Serviço para acessar dados analíticos"""
    
    def __init__(self, connection_string: str):
        self.engine = create_engine(connection_string)
    
    def get_sales_summary(
        self, 
        start_date: datetime = None, 
        end_date: datetime = None,
        group_by: str = 'day'
    ) -> List[SalesSummary]:
        """Obtém resumo de vendas"""
        
        if not start_date:
            start_date = datetime.now() - timedelta(days=30)
        if not end_date:
            end_date = datetime.now()
        
        if group_by == 'day':
            date_format = "DATE_TRUNC('day', data_venda)"
        elif group_by == 'week':
            date_format = "DATE_TRUNC('week', data_venda)"
        elif group_by == 'month':
            date_format = "DATE_TRUNC('month', data_venda)"
        else:
            date_format = "DATE_TRUNC('day', data_venda)"
        
        query = f"""
        SELECT 
            {date_format} as date,
            SUM(valor_total) as total_sales,
            COUNT(*) as quantity,
            AVG(valor_total) as avg_ticket
        FROM analytics.fact_vendas
        WHERE data_venda BETWEEN :start_date AND :end_date
        GROUP BY {date_format}
        ORDER BY date
        """
        
        df = pd.read_sql(
            query, 
            self.engine, 
            params={'start_date': start_date, 'end_date': end_date}
        )
        
        return [SalesSummary(**row) for row in df.to_dict('records')]
    
    def get_customer_metrics(self, customer_id: int = None) -> List[CustomerMetrics]:
        """Obtém métricas de clientes"""
        
        if customer_id:
            query = """
            SELECT 
                c.customer_id,
                SUM(f.valor_total) as total_spent,
                COUNT(*) as total_orders,
                AVG(f.valor_total) as avg_order_value,
                MAX(f.data_venda) as last_order_date
            FROM analytics.fact_vendas f
            JOIN analytics.dim_customer c ON f.customer_id = c.customer_id
            WHERE c.customer_id = :customer_id
            GROUP BY c.customer_id
            """
            params = {'customer_id': customer_id}
        else:
            query = """
            SELECT 
                c.customer_id,
                SUM(f.valor_total) as total_spent,
                COUNT(*) as total_orders,
                AVG(f.valor_total) as avg_order_value,
                MAX(f.data_venda) as last_order_date
            FROM analytics.fact_vendas f
            JOIN analytics.dim_customer c ON f.customer_id = c.customer_id
            GROUP BY c.customer_id
            ORDER BY total_spent DESC
            LIMIT 100
            """
            params = {}
        
        df = pd.read_sql(query, self.engine, params=params)
        return [CustomerMetrics(**row) for row in df.to_dict('records')]
    
    def get_product_performance(
        self, 
        category: str = None,
        limit: int = 50
    ) -> List[ProductPerformance]:
        """Obtém performance de produtos"""
        
        query = """
        SELECT 
            p.product_id,
            p.nome as product_name,
            SUM(f.valor_total) as total_revenue,
            SUM(f.quantidade) as units_sold,
            AVG(f.valor_total / f.quantidade) as avg_price
        FROM analytics.fact_vendas f
        JOIN analytics.dim_product p ON f.product_id = p.product_id
        """
        
        params = {}
        if category:
            query += " WHERE p.categoria = :category"
            params['category'] = category
        
        query += """
        GROUP BY p.product_id, p.nome
        ORDER BY total_revenue DESC
        LIMIT :limit
        """
        params['limit'] = limit
        
        df = pd.read_sql(query, self.engine, params=params)
        return [ProductPerformance(**row) for row in df.to_dict('records')]
    
    def get_time_series(
        self,
        metric: str,
        start_date: datetime,
        end_date: datetime,
        period: str = 'day'
    ) -> List[Dict]:
        """Obtém série temporal de uma métrica"""
        
        if period == 'day':
            date_format = "DATE_TRUNC('day', data_venda)"
        elif period == 'week':
            date_format = "DATE_TRUNC('week', data_venda)"
        else:
            date_format = "DATE_TRUNC('month', data_venda)"
        
        metric_mapping = {
            'sales': 'SUM(valor_total)',
            'quantity': 'COUNT(*)',
            'customers': 'COUNT(DISTINCT customer_id)'
        }
        
        metric_sql = metric_mapping.get(metric, 'SUM(valor_total)')
        
        query = f"""
        SELECT 
            {date_format} as period,
            {metric_sql} as value
        FROM analytics.fact_vendas
        WHERE data_venda BETWEEN :start_date AND :end_date
        GROUP BY {date_format}
        ORDER BY period
        """
        
        df = pd.read_sql(
            query,
            self.engine,
            params={'start_date': start_date, 'end_date': end_date}
        )
        
        return [
            {'period': str(row['period']), 'value': float(row['value']), 'metric': metric}
            for row in df.to_dict('records')
        ]
```

## Passo 5: Criar Rotas da API

**app/routes/analytics.py:**
```python
from fastapi import APIRouter, Depends, HTTPException, Query
from typing import List, Optional
from datetime import datetime, timedelta
from app.services.data_service import DataService
from app.models.data_models import SalesSummary, CustomerMetrics, ProductPerformance
from app.auth.security import get_current_user

router = APIRouter(prefix="/api/analytics", tags=["analytics"])

def get_data_service() -> DataService:
    """Dependency injection para DataService"""
    # Em produção, usar variáveis de ambiente
    connection_string = "postgresql://user:pass@localhost/analytics_db"
    return DataService(connection_string)

@router.get("/sales/summary", response_model=List[SalesSummary])
async def get_sales_summary(
    start_date: Optional[datetime] = Query(None),
    end_date: Optional[datetime] = Query(None),
    group_by: str = Query('day', regex='^(day|week|month)$'),
    service: DataService = Depends(get_data_service),
    current_user: dict = Depends(get_current_user)
):
    """Obtém resumo de vendas"""
    try:
        return service.get_sales_summary(start_date, end_date, group_by)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/customers/metrics", response_model=List[CustomerMetrics])
async def get_customer_metrics(
    customer_id: Optional[int] = Query(None),
    service: DataService = Depends(get_data_service),
    current_user: dict = Depends(get_current_user)
):
    """Obtém métricas de clientes"""
    try:
        return service.get_customer_metrics(customer_id)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/products/performance", response_model=List[ProductPerformance])
async def get_product_performance(
    category: Optional[str] = Query(None),
    limit: int = Query(50, le=1000),
    service: DataService = Depends(get_data_service),
    current_user: dict = Depends(get_current_user)
):
    """Obtém performance de produtos"""
    try:
        return service.get_product_performance(category, limit)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/timeseries")
async def get_time_series(
    metric: str = Query(..., regex='^(sales|quantity|customers)$'),
    start_date: datetime = Query(datetime.now() - timedelta(days=30)),
    end_date: datetime = Query(datetime.now()),
    period: str = Query('day', regex='^(day|week|month)$'),
    service: DataService = Depends(get_data_service),
    current_user: dict = Depends(get_current_user)
):
    """Obtém série temporal"""
    try:
        return service.get_time_series(metric, start_date, end_date, period)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Passo 6: Implementar Autenticação

**app/auth/security.py:**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

security = HTTPBearer()

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """Cria token JWT"""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    """Verifica token JWT"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        token = credentials.credentials
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        return {"username": username}
    except JWTError:
        raise credentials_exception

def get_current_user(current_user: dict = Depends(verify_token)) -> dict:
    """Obtém usuário atual"""
    return current_user
```

## Passo 7: Criar Aplicação Principal

**app/main.py:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routes import analytics

app = FastAPI(
    title="Data Analytics API",
    description="API para acesso a dados analíticos",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Em produção, especificar origens
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Incluir rotas
app.include_router(analytics.router)

@app.get("/")
async def root():
    return {
        "message": "Data Analytics API",
        "docs": "/docs",
        "health": "/health"
    }

@app.get("/health")
async def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Passo 8: Criar Cliente de Teste

**test_client.py:**
```python
import requests
from datetime import datetime, timedelta

BASE_URL = "http://localhost:8000"

# Obter token (em produção, ter endpoint de login)
# Por enquanto, vamos testar sem autenticação (remover Depends temporariamente)

def test_sales_summary():
    """Testa endpoint de resumo de vendas"""
    response = requests.get(f"{BASE_URL}/api/analytics/sales/summary")
    print(f"Status: {response.status_code}")
    print(f"Data: {response.json()[:3]}")  # Primeiros 3 registros

def test_customer_metrics():
    """Testa endpoint de métricas de clientes"""
    response = requests.get(f"{BASE_URL}/api/analytics/customers/metrics")
    print(f"Status: {response.status_code}")
    print(f"Total customers: {len(response.json())}")

def test_product_performance():
    """Testa endpoint de performance de produtos"""
    response = requests.get(f"{BASE_URL}/api/analytics/products/performance?limit=10")
    print(f"Status: {response.status_code}")
    print(f"Top products: {response.json()[:3]}")

def test_time_series():
    """Testa endpoint de série temporal"""
    end_date = datetime.now()
    start_date = end_date - timedelta(days=7)
    
    response = requests.get(
        f"{BASE_URL}/api/analytics/timeseries",
        params={
            "metric": "sales",
            "start_date": start_date.isoformat(),
            "end_date": end_date.isoformat(),
            "period": "day"
        }
    )
    print(f"Status: {response.status_code}")
    print(f"Time series: {response.json()[:5]}")

if __name__ == "__main__":
    print("Testando API...")
    test_sales_summary()
    test_customer_metrics()
    test_product_performance()
    test_time_series()
```

## Passo 9: Executar Aplicação

```bash
# Instalar dependências
pip install -r requirements.txt

# Executar aplicação
uvicorn app.main:app --reload

# Acessar documentação
# http://localhost:8000/docs
```

## Passo 10: Melhorias e Próximos Passos

1. **Cache:** Implementar Redis para cache de queries frequentes
2. **Rate Limiting:** Limitar requisições por usuário
3. **Paginação:** Adicionar paginação para endpoints que retornam muitos dados
4. **Filtros Avançados:** Adicionar mais opções de filtro
5. **WebSockets:** Para dados em tempo real
6. **GraphQL:** Alternativa ao REST para queries complexas

## Checklist de Conclusão

- [ ] API REST criada
- [ ] Endpoints de analytics implementados
- [ ] Autenticação configurada
- [ ] Serviço de dados funcionando
- [ ] Documentação automática (Swagger)
- [ ] Testes escritos
- [ ] Cliente de teste funcionando
- [ ] Documentação completa
