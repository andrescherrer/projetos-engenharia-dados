# Passo a Passo: Framework de Testes de Qualidade de Dados

## Objetivo
Criar um framework para validar qualidade de dados, comparar origem vs destino e detectar inconsistências automaticamente.

## Pré-requisitos
- Python 3.8+
- Conhecimento de SQL
- Conhecimento básico de testes em Python (pytest)

## Passo 1: Configurar Ambiente

```bash
pip install pytest pandas sqlalchemy great-expectations pytest-cov
```

## Passo 2: Estrutura do Projeto

```
data-quality-framework/
├── src/
│   ├── checks/
│   │   ├── completeness.py
│   │   ├── consistency.py
│   │   ├── accuracy.py
│   │   └── validity.py
│   ├── validators/
│   │   └── data_validator.py
│   └── reports/
│       └── quality_report.py
├── tests/
│   └── test_quality_checks.py
└── config/
    └── quality_rules.yaml
```

## Passo 3: Implementar Checks de Completude

**src/checks/completeness.py:**
```python
import pandas as pd
from typing import Dict, List

class CompletenessChecks:
    """Checks para validar completude dos dados"""
    
    @staticmethod
    def check_null_percentage(df: pd.DataFrame, threshold: float = 0.1) -> Dict:
        """Verifica percentual de valores nulos"""
        null_percentages = (df.isnull().sum() / len(df)) * 100
        failed_columns = null_percentages[null_percentages > threshold * 100]
        
        return {
            'check_name': 'null_percentage',
            'passed': len(failed_columns) == 0,
            'failed_columns': failed_columns.to_dict(),
            'message': f"{len(failed_columns)} colunas com mais de {threshold*100}% de nulos"
        }
    
    @staticmethod
    def check_empty_dataframe(df: pd.DataFrame) -> Dict:
        """Verifica se DataFrame está vazio"""
        return {
            'check_name': 'empty_dataframe',
            'passed': len(df) > 0,
            'message': 'DataFrame está vazio' if len(df) == 0 else 'DataFrame contém dados'
        }
    
    @staticmethod
    def check_required_columns(df: pd.DataFrame, required_cols: List[str]) -> Dict:
        """Verifica se colunas obrigatórias existem"""
        missing_cols = set(required_cols) - set(df.columns)
        
        return {
            'check_name': 'required_columns',
            'passed': len(missing_cols) == 0,
            'missing_columns': list(missing_cols),
            'message': f"Colunas faltando: {missing_cols}" if missing_cols else "Todas as colunas obrigatórias presentes"
        }
```

## Passo 4: Implementar Checks de Consistência

**src/checks/consistency.py:**
```python
import pandas as pd
from typing import Dict, List

class ConsistencyChecks:
    """Checks para validar consistência dos dados"""
    
    @staticmethod
    def check_row_count_match(df_source: pd.DataFrame, df_target: pd.DataFrame, tolerance: float = 0.01) -> Dict:
        """Compara contagem de linhas entre origem e destino"""
        source_count = len(df_source)
        target_count = len(df_target)
        difference = abs(source_count - target_count)
        tolerance_count = source_count * tolerance
        
        return {
            'check_name': 'row_count_match',
            'passed': difference <= tolerance_count,
            'source_count': source_count,
            'target_count': target_count,
            'difference': difference,
            'message': f"Diferença de {difference} linhas (tolerância: {tolerance_count})"
        }
    
    @staticmethod
    def check_sum_match(df_source: pd.DataFrame, df_target: pd.DataFrame, column: str) -> Dict:
        """Compara soma de uma coluna entre origem e destino"""
        source_sum = df_source[column].sum()
        target_sum = df_target[column].sum()
        difference = abs(source_sum - target_sum)
        
        return {
            'check_name': 'sum_match',
            'passed': difference < 0.01,  # Tolerância para arredondamento
            'source_sum': source_sum,
            'target_sum': target_sum,
            'difference': difference,
            'message': f"Diferença na soma: {difference}"
        }
    
    @staticmethod
    def check_duplicates(df: pd.DataFrame, columns: List[str] = None) -> Dict:
        """Verifica duplicatas"""
        if columns is None:
            columns = df.columns.tolist()
        
        duplicates = df.duplicated(subset=columns).sum()
        
        return {
            'check_name': 'duplicates',
            'passed': duplicates == 0,
            'duplicate_count': duplicates,
            'message': f"Encontradas {duplicates} linhas duplicadas"
        }
```

## Passo 5: Implementar Checks de Validade

**src/checks/validity.py:**
```python
import pandas as pd
from typing import Dict, List, Callable
from datetime import datetime

class ValidityChecks:
    """Checks para validar regras de negócio"""
    
    @staticmethod
    def check_value_range(df: pd.DataFrame, column: str, min_val: float = None, max_val: float = None) -> Dict:
        """Verifica se valores estão dentro de um range"""
        if min_val is not None:
            below_min = (df[column] < min_val).sum()
        else:
            below_min = 0
        
        if max_val is not None:
            above_max = (df[column] > max_val).sum()
        else:
            above_max = 0
        
        failed_count = below_min + above_max
        
        return {
            'check_name': 'value_range',
            'passed': failed_count == 0,
            'failed_count': failed_count,
            'below_min': below_min,
            'above_max': above_max,
            'message': f"{failed_count} valores fora do range [{min_val}, {max_val}]"
        }
    
    @staticmethod
    def check_allowed_values(df: pd.DataFrame, column: str, allowed_values: List) -> Dict:
        """Verifica se valores estão em uma lista permitida"""
        invalid_values = ~df[column].isin(allowed_values)
        failed_count = invalid_values.sum()
        
        return {
            'check_name': 'allowed_values',
            'passed': failed_count == 0,
            'failed_count': failed_count,
            'invalid_values': df[invalid_values][column].unique().tolist(),
            'message': f"{failed_count} valores não permitidos encontrados"
        }
    
    @staticmethod
    def check_date_format(df: pd.DataFrame, column: str, date_format: str = '%Y-%m-%d') -> Dict:
        """Verifica formato de data"""
        try:
            pd.to_datetime(df[column], format=date_format, errors='raise')
            failed_count = 0
        except:
            failed_count = df[column].isna().sum() + (df[column].apply(lambda x: pd.to_datetime(x, format=date_format, errors='coerce') is None)).sum()
        
        return {
            'check_name': 'date_format',
            'passed': failed_count == 0,
            'failed_count': failed_count,
            'message': f"{failed_count} datas com formato inválido"
        }
    
    @staticmethod
    def check_custom_rule(df: pd.DataFrame, rule_func: Callable, rule_name: str) -> Dict:
        """Aplica uma regra customizada"""
        try:
            result = rule_func(df)
            passed = result if isinstance(result, bool) else result.get('passed', False)
            message = result.get('message', '') if isinstance(result, dict) else ''
            
            return {
                'check_name': f'custom_{rule_name}',
                'passed': passed,
                'message': message
            }
        except Exception as e:
            return {
                'check_name': f'custom_{rule_name}',
                'passed': False,
                'message': f"Erro ao executar regra: {str(e)}"
            }
```

## Passo 6: Criar Validador Principal

**src/validators/data_validator.py:**
```python
import pandas as pd
from typing import List, Dict
from src.checks.completeness import CompletenessChecks
from src.checks.consistency import ConsistencyChecks
from src.checks.validity import ValidityChecks

class DataValidator:
    """Validador principal que executa todos os checks"""
    
    def __init__(self):
        self.completeness = CompletenessChecks()
        self.consistency = ConsistencyChecks()
        self.validity = ValidityChecks()
        self.results = []
    
    def validate_completeness(self, df: pd.DataFrame, required_cols: List[str] = None) -> List[Dict]:
        """Executa todos os checks de completude"""
        results = []
        
        # Check empty
        results.append(self.completeness.check_empty_dataframe(df))
        
        # Check nulls
        results.append(self.completeness.check_null_percentage(df))
        
        # Check required columns
        if required_cols:
            results.append(self.completeness.check_required_columns(df, required_cols))
        
        return results
    
    def validate_consistency(self, df_source: pd.DataFrame, df_target: pd.DataFrame) -> List[Dict]:
        """Executa checks de consistência entre origem e destino"""
        results = []
        
        results.append(self.consistency.check_row_count_match(df_source, df_target))
        
        # Comparar soma de colunas numéricas
        numeric_cols = df_source.select_dtypes(include=['number']).columns
        for col in numeric_cols:
            if col in df_target.columns:
                results.append(self.consistency.check_sum_match(df_source, df_target, col))
        
        return results
    
    def validate_validity(self, df: pd.DataFrame, rules: Dict) -> List[Dict]:
        """Executa checks de validade baseados em regras"""
        results = []
        
        for column, column_rules in rules.items():
            if 'range' in column_rules:
                min_val = column_rules['range'].get('min')
                max_val = column_rules['range'].get('max')
                results.append(self.validity.check_value_range(df, column, min_val, max_val))
            
            if 'allowed_values' in column_rules:
                results.append(self.validity.check_allowed_values(df, column, column_rules['allowed_values']))
            
            if 'date_format' in column_rules:
                results.append(self.validity.check_date_format(df, column, column_rules['date_format']))
        
        return results
    
    def run_all_checks(self, df: pd.DataFrame, df_source: pd.DataFrame = None, 
                      required_cols: List[str] = None, rules: Dict = None) -> Dict:
        """Executa todos os checks"""
        all_results = []
        
        # Completeness
        all_results.extend(self.validate_completeness(df, required_cols))
        
        # Consistency (se temos origem)
        if df_source is not None:
            all_results.extend(self.validate_consistency(df_source, df))
        
        # Validity (se temos regras)
        if rules:
            all_results.extend(self.validate_validity(df, rules))
        
        # Resumo
        passed = sum(1 for r in all_results if r['passed'])
        total = len(all_results)
        
        return {
            'total_checks': total,
            'passed_checks': passed,
            'failed_checks': total - passed,
            'success_rate': passed / total if total > 0 else 0,
            'results': all_results
        }
```

## Passo 7: Criar Relatório de Qualidade

**src/reports/quality_report.py:**
```python
from typing import Dict
import json
from datetime import datetime

class QualityReport:
    """Gera relatórios de qualidade de dados"""
    
    @staticmethod
    def generate_text_report(validation_results: Dict) -> str:
        """Gera relatório em texto"""
        report = []
        report.append("=" * 60)
        report.append("RELATÓRIO DE QUALIDADE DE DADOS")
        report.append("=" * 60)
        report.append(f"Data: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append(f"Total de Checks: {validation_results['total_checks']}")
        report.append(f"Checks Aprovados: {validation_results['passed_checks']}")
        report.append(f"Checks Falhados: {validation_results['failed_checks']}")
        report.append(f"Taxa de Sucesso: {validation_results['success_rate']*100:.2f}%")
        report.append("")
        report.append("-" * 60)
        report.append("DETALHES DOS CHECKS")
        report.append("-" * 60)
        
        for result in validation_results['results']:
            status = "✅ PASSOU" if result['passed'] else "❌ FALHOU"
            report.append(f"\n{status} - {result['check_name']}")
            report.append(f"   {result.get('message', '')}")
            if not result['passed'] and 'failed_count' in result:
                report.append(f"   Registros com problema: {result['failed_count']}")
        
        return "\n".join(report)
    
    @staticmethod
    def generate_json_report(validation_results: Dict) -> str:
        """Gera relatório em JSON"""
        return json.dumps(validation_results, indent=2, default=str)
    
    @staticmethod
    def save_report(validation_results: Dict, output_path: str, format: str = 'text'):
        """Salva relatório em arquivo"""
        if format == 'text':
            content = QualityReport.generate_text_report(validation_results)
        else:
            content = QualityReport.generate_json_report(validation_results)
        
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(content)
        
        print(f"✅ Relatório salvo em {output_path}")
```

## Passo 8: Criar Testes

**tests/test_quality_checks.py:**
```python
import pytest
import pandas as pd
from src.validators.data_validator import DataValidator

def test_completeness_checks():
    """Testa checks de completude"""
    validator = DataValidator()
    
    # DataFrame válido
    df_valid = pd.DataFrame({
        'id': [1, 2, 3],
        'nome': ['A', 'B', 'C'],
        'valor': [10, 20, 30]
    })
    
    results = validator.validate_completeness(df_valid, required_cols=['id', 'nome'])
    assert all(r['passed'] for r in results)
    
    # DataFrame com muitos nulos
    df_invalid = pd.DataFrame({
        'id': [1, 2, None, None, None],
        'nome': ['A', None, None, None, None]
    })
    
    results = validator.validate_completeness(df_invalid)
    assert not all(r['passed'] for r in results)

def test_consistency_checks():
    """Testa checks de consistência"""
    validator = DataValidator()
    
    df_source = pd.DataFrame({'valor': [10, 20, 30]})
    df_target = pd.DataFrame({'valor': [10, 20, 30]})
    
    results = validator.validate_consistency(df_source, df_target)
    assert all(r['passed'] for r in results)

if __name__ == "__main__":
    pytest.main([__file__, '-v'])
```

## Passo 9: Exemplo de Uso

**exemplo_uso.py:**
```python
import pandas as pd
from src.validators.data_validator import DataValidator
from src.reports.quality_report import QualityReport

# Dados de exemplo
df_source = pd.DataFrame({
    'id': [1, 2, 3, 4, 5],
    'nome': ['A', 'B', 'C', 'D', 'E'],
    'idade': [25, 30, 35, 40, 45],
    'categoria': ['X', 'Y', 'X', 'Y', 'X']
})

df_target = pd.DataFrame({
    'id': [1, 2, 3, 4, 5],
    'nome': ['A', 'B', 'C', 'D', 'E'],
    'idade': [25, 30, 35, 40, 45],
    'categoria': ['X', 'Y', 'X', 'Y', 'X']
})

# Regras de validação
rules = {
    'idade': {
        'range': {'min': 18, 'max': 100}
    },
    'categoria': {
        'allowed_values': ['X', 'Y', 'Z']
    }
}

# Executar validação
validator = DataValidator()
results = validator.run_all_checks(
    df=df_target,
    df_source=df_source,
    required_cols=['id', 'nome', 'idade'],
    rules=rules
)

# Gerar relatório
report = QualityReport.generate_text_report(results)
print(report)

# Salvar relatório
QualityReport.save_report(results, 'quality_report.txt')
```

## Passo 10: Integrar com Pipeline

Para integrar com seu pipeline existente:

```python
from src.validators.data_validator import DataValidator

def pipeline_with_quality_check():
    # ... seu pipeline ...
    
    # Após transformação
    validator = DataValidator()
    results = validator.run_all_checks(df_transformed, df_source, rules=quality_rules)
    
    if results['success_rate'] < 0.95:  # 95% de sucesso mínimo
        raise ValueError("Qualidade de dados abaixo do esperado")
    
    # Continuar pipeline...
```

## Checklist de Conclusão

- [ ] Ambiente configurado
- [ ] Checks de completude implementados
- [ ] Checks de consistência implementados
- [ ] Checks de validade implementados
- [ ] Validador principal criado
- [ ] Sistema de relatórios funcionando
- [ ] Testes escritos e passando
- [ ] Framework integrado com pipeline
- [ ] Documentação completa
