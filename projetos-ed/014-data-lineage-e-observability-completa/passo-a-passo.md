# Passo a Passo: Data Lineage e Observability Completa

## Objetivo
Implementar sistema completo de linhagem de dados e observability para rastrear impacto de mudanças e monitorar saúde de pipelines.

## Pré-requisitos
- Docker
- Python 3.8+
- Conhecimento de grafos e métricas

## Passo 1: Configurar OpenLineage

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  marquez:
    image: marquezproject/marquez:latest
    ports:
      - "5000:5000"
    environment:
      - MARQUEZ_CONFIG=marquez.dev.yml
    volumes:
      - ./marquez.yml:/etc/marquez/marquez.dev.yml

  postgres-lineage:
    image: postgres:13
    environment:
      POSTGRES_USER: marquez
      POSTGRES_PASSWORD: marquez
      POSTGRES_DB: marquez
    volumes:
      - lineage-data:/var/lib/postgresql/data
```

## Passo 2: Coletor de Linhagem

**src/lineage/lineage_collector.py:**
```python
from openlineage.client import OpenLineageClient
from openlineage.client.run import Run, Job, Dataset
from typing import List, Dict
from datetime import datetime

class LineageCollector:
    """Coleta linhagem de dados de pipelines"""
    
    def __init__(self, namespace: str = "default"):
        self.client = OpenLineageClient()
        self.namespace = namespace
    
    def track_pipeline_run(
        self,
        pipeline_name: str,
        inputs: List[Dict],
        outputs: List[Dict],
        run_id: str = None
    ):
        """Rastreia execução de pipeline"""
        
        if not run_id:
            run_id = f"{pipeline_name}-{datetime.now().isoformat()}"
        
        # Criar datasets de entrada
        input_datasets = [
            Dataset(
                namespace=self.namespace,
                name=input["name"],
                facets=input.get("facets", {})
            )
            for input in inputs
        ]
        
        # Criar datasets de saída
        output_datasets = [
            Dataset(
                namespace=self.namespace,
                name=output["name"],
                facets=output.get("facets", {})
            )
            for output in outputs
        ]
        
        # Criar job
        job = Job(
            namespace=self.namespace,
            name=pipeline_name
        )
        
        # Criar run
        run = Run(
            runId=run_id,
            facets={}
        )
        
        # Enviar evento de início
        self.client.start_run(
            run=run,
            job=job,
            inputs=input_datasets
        )
        
        # Enviar evento de conclusão
        self.client.complete_run(
            run=run,
            job=job,
            outputs=output_datasets
        )
        
        print(f"✅ Linhagem rastreada: {pipeline_name}")

# Exemplo de uso
collector = LineageCollector()

collector.track_pipeline_run(
    pipeline_name="daily_sales_etl",
    inputs=[
        {"name": "erp_db.sales", "facets": {"schema": {"fields": ["id", "amount"]}}}
    ],
    outputs=[
        {"name": "analytics_db.fact_sales", "facets": {}}
    ]
)
```

## Passo 3: Impact Analysis

**src/lineage/impact_analyzer.py:**
```python
from typing import List, Dict, Set
from collections import defaultdict

class ImpactAnalyzer:
    """Analisa impacto de mudanças"""
    
    def __init__(self):
        self.lineage_graph = defaultdict(set)  # downstream -> upstream
    
    def build_lineage_graph(self, lineage_data: List[Dict]):
        """Constrói grafo de linhagem"""
        for item in lineage_data:
            upstream = item["upstream"]
            downstream = item["downstream"]
            self.lineage_graph[downstream].add(upstream)
    
    def find_downstream_impact(self, dataset: str) -> Set[str]:
        """Encontra todos os datasets downstream"""
        impacted = set()
        to_process = [dataset]
        
        while to_process:
            current = to_process.pop()
            if current in impacted:
                continue
            
            impacted.add(current)
            
            # Encontrar todos que dependem de current
            for downstream, upstreams in self.lineage_graph.items():
                if current in upstreams:
                    to_process.append(downstream)
        
        return impacted - {dataset}
    
    def find_upstream_dependencies(self, dataset: str) -> Set[str]:
        """Encontra todas as dependências upstream"""
        dependencies = set()
        to_process = [dataset]
        
        while to_process:
            current = to_process.pop()
            if current in dependencies:
                continue
            
            dependencies.add(current)
            
            # Encontrar upstreams de current
            if current in self.lineage_graph:
                upstreams = self.lineage_graph[current]
                to_process.extend(upstreams)
        
        return dependencies - {dataset}

# Exemplo
analyzer = ImpactAnalyzer()

lineage_data = [
    {"upstream": "source.table1", "downstream": "staging.table1"},
    {"upstream": "staging.table1", "downstream": "analytics.fact_sales"},
    {"upstream": "analytics.fact_sales", "downstream": "reports.sales_report"}
]

analyzer.build_lineage_graph(lineage_data)

# Se mudarmos staging.table1, o que é impactado?
impacted = analyzer.find_downstream_impact("staging.table1")
print(f"Datasets impactados: {impacted}")
```

## Passo 4: Métricas e Observability

**src/observability/metrics_collector.py:**
```python
from prometheus_client import Counter, Histogram, Gauge
from typing import Dict
import time

class PipelineMetrics:
    """Coleta métricas de pipelines"""
    
    def __init__(self):
        self.pipeline_runs = Counter(
            'pipeline_runs_total',
            'Total pipeline runs',
            ['pipeline_name', 'status']
        )
        
        self.pipeline_duration = Histogram(
            'pipeline_duration_seconds',
            'Pipeline execution duration',
            ['pipeline_name']
        )
        
        self.pipeline_rows_processed = Counter(
            'pipeline_rows_processed_total',
            'Total rows processed',
            ['pipeline_name']
        )
        
        self.pipeline_status = Gauge(
            'pipeline_status',
            'Pipeline status (1=running, 0=stopped)',
            ['pipeline_name']
        )
    
    def record_pipeline_start(self, pipeline_name: str):
        """Registra início do pipeline"""
        self.pipeline_status.labels(pipeline_name=pipeline_name).set(1)
    
    def record_pipeline_end(
        self,
        pipeline_name: str,
        status: str,
        duration: float,
        rows_processed: int = 0
    ):
        """Registra fim do pipeline"""
        self.pipeline_status.labels(pipeline_name=pipeline_name).set(0)
        self.pipeline_runs.labels(
            pipeline_name=pipeline_name,
            status=status
        ).inc()
        self.pipeline_duration.labels(
            pipeline_name=pipeline_name
        ).observe(duration)
        self.pipeline_rows_processed.labels(
            pipeline_name=pipeline_name
        ).inc(rows_processed)

# Decorator para métricas
def track_pipeline_metrics(metrics: PipelineMetrics):
    def decorator(func):
        def wrapper(*args, **kwargs):
            pipeline_name = kwargs.get('pipeline_name', func.__name__)
            metrics.record_pipeline_start(pipeline_name)
            
            start_time = time.time()
            try:
                result = func(*args, **kwargs)
                status = "success"
                rows = result.get('rows_processed', 0) if isinstance(result, dict) else 0
            except Exception as e:
                status = "failed"
                rows = 0
                raise
            finally:
                duration = time.time() - start_time
                metrics.record_pipeline_end(pipeline_name, status, duration, rows)
            
            return result
        return wrapper
    return decorator
```

## Passo 5: Alertas Inteligentes

**src/observability/alerting.py:**
```python
from typing import Dict, List
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class Alert:
    severity: str  # critical, warning, info
    message: str
    pipeline_name: str
    timestamp: datetime
    metadata: Dict

class AlertManager:
    """Gerencia alertas de pipelines"""
    
    def __init__(self):
        self.alert_rules = []
        self.alert_history = []
    
    def add_alert_rule(self, rule: Dict):
        """Adiciona regra de alerta"""
        self.alert_rules.append(rule)
    
    def check_alerts(self, pipeline_metrics: Dict) -> List[Alert]:
        """Verifica regras e gera alertas"""
        alerts = []
        
        for rule in self.alert_rules:
            if self._evaluate_rule(rule, pipeline_metrics):
                alert = Alert(
                    severity=rule["severity"],
                    message=rule["message"],
                    pipeline_name=pipeline_metrics["name"],
                    timestamp=datetime.now(),
                    metadata=rule.get("metadata", {})
                )
                alerts.append(alert)
                self.alert_history.append(alert)
        
        return alerts
    
    def _evaluate_rule(self, rule: Dict, metrics: Dict) -> bool:
        """Avalia se regra deve gerar alerta"""
        condition = rule["condition"]
        
        if condition["type"] == "threshold":
            value = metrics.get(condition["metric"])
            threshold = condition["threshold"]
            operator = condition["operator"]
            
            if operator == ">":
                return value > threshold
            elif operator == "<":
                return value < threshold
            elif operator == "==":
                return value == threshold
        
        elif condition["type"] == "anomaly":
            # Detecção de anomalias usando estatísticas
            return self._detect_anomaly(condition, metrics)
        
        return False
    
    def _detect_anomaly(self, condition: Dict, metrics: Dict) -> bool:
        """Detecta anomalias usando desvio padrão"""
        # Implementação simplificada
        # Em produção, usar ML para detecção de anomalias
        return False
    
    def send_alert(self, alert: Alert):
        """Envia alerta (email, Slack, etc.)"""
        print(f"🚨 ALERTA [{alert.severity.upper()}]: {alert.message}")
        # Implementar envio real

# Exemplo
alert_manager = AlertManager()

alert_manager.add_alert_rule({
    "severity": "critical",
    "message": "Pipeline falhou",
    "condition": {
        "type": "threshold",
        "metric": "status",
        "operator": "==",
        "threshold": "failed"
    }
})

alert_manager.add_alert_rule({
    "severity": "warning",
    "message": "Pipeline demorou mais que o esperado",
    "condition": {
        "type": "threshold",
        "metric": "duration_seconds",
        "operator": ">",
        "threshold": 3600
    }
})
```

## Checklist de Conclusão

- [ ] OpenLineage configurado
- [ ] Coletor de linhagem implementado
- [ ] Impact analysis funcionando
- [ ] Métricas coletadas
- [ ] Grafana/Prometheus configurado
- [ ] Sistema de alertas implementado
- [ ] Dashboard de observability
- [ ] Documentação completa
