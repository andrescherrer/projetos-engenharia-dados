# Passo a Passo: ETL Básico com Arquivos Locais

## Objetivo
Criar um pipeline ETL simples que processa arquivos CSV/JSON locais, aplica transformações básicas e salva o resultado em outro formato.

## Pré-requisitos
- Python 3.8+ instalado
- Conhecimento básico de Python
- Editor de código (VS Code, PyCharm, etc.)

## Passo 1: Configurar Ambiente

```bash
# Criar ambiente virtual
python -m venv venv

# Ativar ambiente virtual
# Linux/Mac:
source venv/bin/activate
# Windows:
venv\Scripts\activate

# Instalar dependências
pip install pandas polars
```

## Passo 2: Estrutura do Projeto

```
projeto-etl/
├── data/
│   ├── input/          # Arquivos de entrada
│   └── output/         # Arquivos processados
├── src/
│   ├── extract.py      # Função de extração
│   ├── transform.py    # Função de transformação
│   └── load.py         # Função de carga
├── main.py             # Script principal
└── requirements.txt
```

## Passo 3: Implementar Extract (Extração)

**src/extract.py:**
```python
import pandas as pd
import json
from pathlib import Path

def extract_csv(file_path: str) -> pd.DataFrame:
    """Extrai dados de um arquivo CSV"""
    try:
        df = pd.read_csv(file_path)
        print(f"✅ Extraído {len(df)} linhas de {file_path}")
        return df
    except Exception as e:
        print(f"❌ Erro ao extrair CSV: {e}")
        raise

def extract_json(file_path: str) -> pd.DataFrame:
    """Extrai dados de um arquivo JSON"""
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        df = pd.DataFrame(data)
        print(f"✅ Extraído {len(df)} linhas de {file_path}")
        return df
    except Exception as e:
        print(f"❌ Erro ao extrair JSON: {e}")
        raise

def extract_all_files(input_dir: str) -> list[pd.DataFrame]:
    """Extrai todos os arquivos de um diretório"""
    dataframes = []
    path = Path(input_dir)
    
    for file in path.iterdir():
        if file.suffix == '.csv':
            df = extract_csv(str(file))
            dataframes.append(df)
        elif file.suffix == '.json':
            df = extract_json(str(file))
            dataframes.append(df)
    
    return dataframes
```

## Passo 4: Implementar Transform (Transformação)

**src/transform.py:**
```python
import pandas as pd

def clean_data(df: pd.DataFrame) -> pd.DataFrame:
    """Remove linhas duplicadas e valores nulos"""
    df_clean = df.drop_duplicates()
    df_clean = df_clean.dropna()
    return df_clean

def standardize_columns(df: pd.DataFrame) -> pd.DataFrame:
    """Padroniza nomes de colunas (lowercase, sem espaços)"""
    df.columns = df.columns.str.lower().str.replace(' ', '_')
    return df

def add_processed_date(df: pd.DataFrame) -> pd.DataFrame:
    """Adiciona coluna com data de processamento"""
    from datetime import datetime
    df['processed_at'] = datetime.now()
    return df

def transform(df: pd.DataFrame) -> pd.DataFrame:
    """Aplica todas as transformações"""
    df = clean_data(df)
    df = standardize_columns(df)
    df = add_processed_date(df)
    print(f"✅ Transformado: {len(df)} linhas")
    return df
```

## Passo 5: Implementar Load (Carga)

**src/load.py:**
```python
import pandas as pd
from pathlib import Path

def load_to_csv(df: pd.DataFrame, output_path: str):
    """Salva DataFrame em CSV"""
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    df.to_csv(output_path, index=False)
    print(f"✅ Salvo em {output_path}")

def load_to_parquet(df: pd.DataFrame, output_path: str):
    """Salva DataFrame em Parquet (formato otimizado)"""
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    df.to_parquet(output_path, index=False)
    print(f"✅ Salvo em {output_path}")

def load_to_json(df: pd.DataFrame, output_path: str):
    """Salva DataFrame em JSON"""
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    df.to_json(output_path, orient='records', indent=2)
    print(f"✅ Salvo em {output_path}")
```

## Passo 6: Criar Script Principal

**main.py:**
```python
from src.extract import extract_all_files
from src.transform import transform
from src.load import load_to_csv, load_to_parquet
import pandas as pd

def main():
    # Extract
    print("🔄 Iniciando extração...")
    dataframes = extract_all_files('data/input')
    
    # Transform e Load
    for i, df in enumerate(dataframes):
        print(f"\n📊 Processando arquivo {i+1}...")
        
        # Transform
        df_transformed = transform(df)
        
        # Load
        output_csv = f'data/output/arquivo_{i+1}.csv'
        output_parquet = f'data/output/arquivo_{i+1}.parquet'
        
        load_to_csv(df_transformed, output_csv)
        load_to_parquet(df_transformed, output_parquet)
    
    print("\n✅ Pipeline concluído com sucesso!")

if __name__ == "__main__":
    main()
```

## Passo 7: Criar Dados de Teste

**Criar data/input/exemplo.csv:**
```csv
nome,idade,cidade
João,25,São Paulo
Maria,30,Rio de Janeiro
Pedro,22,Belo Horizonte
Ana,28,Curitiba
```

**Criar data/input/exemplo.json:**
```json
[
  {"produto": "Notebook", "preco": 3500, "estoque": 10},
  {"produto": "Mouse", "preco": 50, "estoque": 100},
  {"produto": "Teclado", "preco": 150, "estoque": 50}
]
```

## Passo 8: Executar o Pipeline

```bash
python main.py
```

## Passo 9: Melhorias e Próximos Passos

### Adicionar Logging
```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
```

### Adicionar Tratamento de Erros
```python
try:
    df = extract_csv('data/input/arquivo.csv')
except FileNotFoundError:
    logging.error("Arquivo não encontrado")
except Exception as e:
    logging.error(f"Erro inesperado: {e}")
```

### Adicionar Validação de Dados
```python
def validate_data(df: pd.DataFrame) -> bool:
    """Valida se os dados estão corretos"""
    if df.empty:
        raise ValueError("DataFrame está vazio")
    # Adicionar mais validações conforme necessário
    return True
```

## Exercícios Práticos

1. **Processar múltiplos arquivos:** Modifique o código para processar todos os arquivos de uma pasta
2. **Adicionar mais transformações:** Crie funções para calcular estatísticas, filtrar dados, etc.
3. **Suportar mais formatos:** Adicione suporte para Excel (.xlsx)
4. **Criar relatório:** Gere um relatório em texto com estatísticas dos dados processados

## Recursos de Aprendizado

- [Documentação Pandas](https://pandas.pydata.org/docs/)
- [Documentação Polars](https://pola-rs.github.io/polars/)
- [Python Pathlib](https://docs.python.org/3/library/pathlib.html)

## Checklist de Conclusão

- [ ] Ambiente configurado
- [ ] Funções Extract implementadas
- [ ] Funções Transform implementadas
- [ ] Funções Load implementadas
- [ ] Script principal funcionando
- [ ] Dados de teste criados
- [ ] Pipeline executado com sucesso
- [ ] Código documentado
- [ ] Tratamento de erros implementado
