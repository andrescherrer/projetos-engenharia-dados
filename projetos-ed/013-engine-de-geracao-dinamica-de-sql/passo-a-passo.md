# Passo a Passo: Engine de Geração Dinâmica de SQL

## Objetivo
Criar um engine que gera SQL dinamicamente a partir de metadados, reduzindo código repetitivo e automatizando criação de queries.

## Pré-requisitos
- Python 3.8+
- Conhecimento avançado de SQL e Python
- Jinja2 para templates

## Passo 1: Estrutura do Projeto

```
sql-engine/
├── src/
│   ├── generators/
│   │   ├── select_generator.py
│   │   ├── insert_generator.py
│   │   └── update_generator.py
│   ├── templates/
│   │   └── sql_templates.j2
│   └── metadata/
│       └── schema_registry.py
└── tests/
```

## Passo 2: Criar Registry de Metadados

**src/metadata/schema_registry.py:**
```python
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class Column:
    name: str
    data_type: str
    nullable: bool = True
    primary_key: bool = False
    foreign_key: Dict = None

@dataclass
class Table:
    name: str
    schema: str
    columns: List[Column]
    description: str = ""

class SchemaRegistry:
    """Registry de schemas de tabelas"""
    
    def __init__(self):
        self.tables: Dict[str, Table] = {}
    
    def register_table(self, table: Table):
        """Registra uma tabela"""
        key = f"{table.schema}.{table.name}"
        self.tables[key] = table
    
    def get_table(self, schema: str, table: str) -> Table:
        """Obtém tabela do registry"""
        key = f"{schema}.{table}"
        return self.tables.get(key)
    
    def get_columns(self, schema: str, table: str) -> List[Column]:
        """Obtém colunas de uma tabela"""
        table_obj = self.get_table(schema, table)
        return table_obj.columns if table_obj else []

# Exemplo de uso
registry = SchemaRegistry()

# Registrar tabela
vendas_table = Table(
    name="fact_vendas",
    schema="analytics",
    columns=[
        Column("fact_id", "INTEGER", primary_key=True),
        Column("date_id", "INTEGER", foreign_key={"table": "dim_date", "column": "date_id"}),
        Column("customer_id", "INTEGER"),
        Column("valor_total", "DECIMAL(10,2)"),
        Column("quantidade", "INTEGER")
    ],
    description="Tabela fato de vendas"
)

registry.register_table(vendas_table)
```

## Passo 3: Gerador de SELECT

**src/generators/select_generator.py:**
```python
from typing import List, Dict, Optional
from jinja2 import Template
from src.metadata.schema_registry import SchemaRegistry

class SelectGenerator:
    """Gera queries SELECT dinamicamente"""
    
    def __init__(self, registry: SchemaRegistry):
        self.registry = registry
    
    def generate_select(
        self,
        schema: str,
        table: str,
        columns: Optional[List[str]] = None,
        filters: Optional[Dict[str, any]] = None,
        order_by: Optional[List[str]] = None,
        limit: Optional[int] = None,
        joins: Optional[List[Dict]] = None
    ) -> str:
        """Gera SELECT statement"""
        
        table_obj = self.registry.get_table(schema, table)
        if not table_obj:
            raise ValueError(f"Tabela {schema}.{table} não encontrada")
        
        # Colunas
        if columns is None:
            columns = [col.name for col in table_obj.columns]
        elif columns == ["*"]:
            columns = [col.name for col in table_obj.columns]
        
        # FROM
        from_clause = f"{schema}.{table}"
        
        # JOINs
        join_clauses = []
        if joins:
            for join in joins:
                join_type = join.get("type", "INNER")
                join_table = join["table"]
                join_schema = join.get("schema", schema)
                join_on = join["on"]
                join_clauses.append(
                    f"{join_type} JOIN {join_schema}.{join_table} ON {join_on}"
                )
        
        # WHERE
        where_clauses = []
        if filters:
            for col, value in filters.items():
                if isinstance(value, list):
                    where_clauses.append(f"{col} IN ({', '.join(map(str, value))})")
                elif isinstance(value, dict):
                    operator = value.get("operator", "=")
                    val = value["value"]
                    where_clauses.append(f"{col} {operator} {val}")
                else:
                    where_clauses.append(f"{col} = '{value}'")
        
        # ORDER BY
        order_clause = ""
        if order_by:
            order_clause = f"ORDER BY {', '.join(order_by)}"
        
        # LIMIT
        limit_clause = f"LIMIT {limit}" if limit else ""
        
        # Montar query
        query = f"SELECT {', '.join(columns)}\n"
        query += f"FROM {from_clause}\n"
        
        if join_clauses:
            query += "\n".join(join_clauses) + "\n"
        
        if where_clauses:
            query += f"WHERE {' AND '.join(where_clauses)}\n"
        
        if order_clause:
            query += order_clause + "\n"
        
        if limit_clause:
            query += limit_clause
        
        return query.strip()

# Exemplo de uso
generator = SelectGenerator(registry)

query = generator.generate_select(
    schema="analytics",
    table="fact_vendas",
    columns=["fact_id", "valor_total", "quantidade"],
    filters={"valor_total": {"operator": ">", "value": 100}},
    order_by=["valor_total DESC"],
    limit=100
)

print(query)
```

## Passo 4: Gerador de INSERT

**src/generators/insert_generator.py:**
```python
from typing import Dict, List
from src.metadata.schema_registry import SchemaRegistry

class InsertGenerator:
    """Gera queries INSERT dinamicamente"""
    
    def __init__(self, registry: SchemaRegistry):
        self.registry = registry
    
    def generate_insert(
        self,
        schema: str,
        table: str,
        data: Dict[str, any]
    ) -> str:
        """Gera INSERT statement"""
        
        table_obj = self.registry.get_table(schema, table)
        if not table_obj:
            raise ValueError(f"Tabela {schema}.{table} não encontrada")
        
        # Validar colunas
        valid_columns = {col.name for col in table_obj.columns}
        provided_columns = set(data.keys())
        
        invalid_columns = provided_columns - valid_columns
        if invalid_columns:
            raise ValueError(f"Colunas inválidas: {invalid_columns}")
        
        columns = list(data.keys())
        values = [self._format_value(data[col]) for col in columns]
        
        query = f"INSERT INTO {schema}.{table} ({', '.join(columns)})\n"
        query += f"VALUES ({', '.join(values)})"
        
        return query
    
    def generate_bulk_insert(
        self,
        schema: str,
        table: str,
        data_list: List[Dict[str, any]]
    ) -> str:
        """Gera INSERT bulk"""
        
        if not data_list:
            raise ValueError("Lista de dados vazia")
        
        table_obj = self.registry.get_table(schema, table)
        columns = list(data_list[0].keys())
        
        values_list = []
        for data in data_list:
            values = [self._format_value(data[col]) for col in columns]
            values_list.append(f"({', '.join(values)})")
        
        query = f"INSERT INTO {schema}.{table} ({', '.join(columns)})\n"
        query += f"VALUES {', '.join(values_list)}"
        
        return query
    
    def _format_value(self, value: any) -> str:
        """Formata valor para SQL"""
        if value is None:
            return "NULL"
        elif isinstance(value, str):
            return f"'{value.replace("'", "''")}'"
        elif isinstance(value, (int, float)):
            return str(value)
        elif isinstance(value, bool):
            return "TRUE" if value else "FALSE"
        else:
            return f"'{str(value)}'"
```

## Passo 5: Gerador de UPDATE

**src/generators/update_generator.py:**
```python
from typing import Dict
from src.metadata.schema_registry import SchemaRegistry

class UpdateGenerator:
    """Gera queries UPDATE dinamicamente"""
    
    def __init__(self, registry: SchemaRegistry):
        self.registry = registry
    
    def generate_update(
        self,
        schema: str,
        table: str,
        set_values: Dict[str, any],
        where_clause: Dict[str, any]
    ) -> str:
        """Gera UPDATE statement"""
        
        table_obj = self.registry.get_table(schema, table)
        if not table_obj:
            raise ValueError(f"Tabela {schema}.{table} não encontrada")
        
        # SET clause
        set_clauses = []
        for col, value in set_values.items():
            formatted_value = self._format_value(value)
            set_clauses.append(f"{col} = {formatted_value}")
        
        # WHERE clause
        where_clauses = []
        for col, value in where_clause.items():
            formatted_value = self._format_value(value)
            where_clauses.append(f"{col} = {formatted_value}")
        
        query = f"UPDATE {schema}.{table}\n"
        query += f"SET {', '.join(set_clauses)}\n"
        query += f"WHERE {' AND '.join(where_clauses)}"
        
        return query
    
    def _format_value(self, value: any) -> str:
        """Formata valor para SQL"""
        if value is None:
            return "NULL"
        elif isinstance(value, str):
            return f"'{value.replace("'", "''")}'"
        else:
            return str(value)
```

## Passo 6: Template Engine com Jinja2

**src/templates/sql_templates.j2:**
```jinja
-- Template para SELECT com agregações
SELECT 
{%- for col in select_columns %}
    {{ col }}{% if not loop.last %},{% endif %}
{%- endfor %}
FROM {{ schema }}.{{ table }}
{% if joins %}
{%- for join in joins %}
{{ join.type }} JOIN {{ join.schema }}.{{ join.table }} ON {{ join.condition }}
{%- endfor %}
{% endif %}
{% if filters %}
WHERE 
{%- for filter in filters %}
    {{ filter.column }} {{ filter.operator }} {{ filter.value }}{% if not loop.last %} AND{% endif %}
{%- endfor %}
{% endif %}
{% if group_by %}
GROUP BY {{ group_by | join(', ') }}
{% endif %}
{% if having %}
HAVING {{ having }}
{% endif %}
{% if order_by %}
ORDER BY {{ order_by | join(', ') }}
{% endif %}
{% if limit %}
LIMIT {{ limit }}
{% endif %}
```

**src/generators/template_generator.py:**
```python
from jinja2 import Environment, FileSystemLoader
from pathlib import Path

class TemplateSQLGenerator:
    """Gera SQL usando templates Jinja2"""
    
    def __init__(self, template_dir: str = "src/templates"):
        self.env = Environment(
            loader=FileSystemLoader(template_dir),
            trim_blocks=True,
            lstrip_blocks=True
        )
    
    def generate_from_template(
        self,
        template_name: str,
        **kwargs
    ) -> str:
        """Gera SQL a partir de template"""
        template = self.env.get_template(template_name)
        return template.render(**kwargs)

# Exemplo
generator = TemplateSQLGenerator()

sql = generator.generate_from_template(
    "sql_templates.j2",
    select_columns=["SUM(valor_total)", "COUNT(*)"],
    schema="analytics",
    table="fact_vendas",
    filters=[
        {"column": "date_id", "operator": ">", "value": "2024-01-01"}
    ],
    group_by=["customer_id"],
    order_by=["SUM(valor_total) DESC"],
    limit=10
)
```

## Checklist de Conclusão

- [ ] Registry de metadados criado
- [ ] Gerador de SELECT implementado
- [ ] Gerador de INSERT implementado
- [ ] Gerador de UPDATE implementado
- [ ] Templates Jinja2 criados
- [ ] Validação de schemas
- [ ] Testes escritos
- [ ] Documentação completa
