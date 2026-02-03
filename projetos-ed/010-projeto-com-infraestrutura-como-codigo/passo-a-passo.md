# Passo a Passo: Projeto com Infraestrutura como Código

## Objetivo
Criar um pipeline completamente reproduzível usando Docker, Terraform e CI/CD, demonstrando boas práticas de infraestrutura como código.

## Pré-requisitos
- Docker instalado
- Terraform instalado
- Conta em cloud provider (AWS/GCP/Azure)
- Conhecimento básico de Linux

## Passo 1: Estrutura do Projeto

```
iac-pipeline/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── scripts/
│   └── deploy.sh
└── README.md
```

## Passo 2: Criar Dockerfile

**docker/Dockerfile:**
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Instalar dependências do sistema
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copiar requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código
COPY . .

# Variáveis de ambiente
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app

# Comando padrão
CMD ["python", "main.py"]
```

## Passo 3: Criar Docker Compose

**docker/docker-compose.yml:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-analytics_db}
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  pipeline:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: ${DB_USER:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-postgres}
      DB_NAME: ${DB_NAME:-analytics_db}
    volumes:
      - ../data:/app/data
      - ../logs:/app/logs
    command: python main.py

  airflow:
    image: apache/airflow:2.7.0
    depends_on:
      - postgres
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://postgres:postgres@postgres/airflow
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - ../dags:/opt/airflow/dags
      - ../logs:/opt/airflow/logs
    ports:
      - "8080:8080"
    command: >
      bash -c "
      airflow db init &&
      airflow users create --username admin --firstname Admin --lastname User --role Admin --email admin@example.com --password admin &&
      airflow webserver & airflow scheduler
      "

volumes:
  postgres-data:
```

## Passo 4: Terraform para AWS

**terraform/main.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "data-pipeline/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "data-pipeline-vpc"
  }
}

# Subnet
resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "${var.aws_region}a"

  tags = {
    Name = "data-pipeline-subnet"
  }
}

# Security Group
resource "aws_security_group" "rds" {
  name        = "rds-security-group"
  description = "Security group for RDS"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "analytics_db" {
  identifier     = "analytics-db"
  engine         = "postgres"
  engine_version = "13.7"
  instance_class = var.db_instance_class

  allocated_storage     = 20
  max_allocated_storage = 100
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "analytics_db"
  username = var.db_username
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  skip_final_snapshot = false
  final_snapshot_identifier = "analytics-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  tags = {
    Name = "analytics-database"
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "main-subnet-group"
  subnet_ids = [aws_subnet.main.id]

  tags = {
    Name = "main-subnet-group"
  }
}

# S3 Bucket para dados
resource "aws_s3_bucket" "data_lake" {
  bucket = "${var.project_name}-data-lake"

  tags = {
    Name = "Data Lake"
  }
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration {
    status = "Enabled"
  }
}

# ECS Cluster para Airflow
resource "aws_ecs_cluster" "airflow" {
  name = "${var.project_name}-airflow-cluster"

  tags = {
    Name = "Airflow Cluster"
  }
}

# ECR Repository para imagens Docker
resource "aws_ecr_repository" "pipeline" {
  name                 = "${var.project_name}-pipeline"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

**terraform/variables.tf:**
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "data-pipeline"
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_username" {
  description = "Database username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```

**terraform/outputs.tf:**
```hcl
output "rds_endpoint" {
  description = "RDS endpoint"
  value       = aws_db_instance.analytics_db.endpoint
}

output "s3_bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.data_lake.bucket
}

output "ecr_repository_url" {
  description = "ECR repository URL"
  value       = aws_ecr_repository.pipeline.repository_url
}
```

## Passo 5: CI/CD com GitHub Actions

**.github/workflows/ci-cd.yml:**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: data-pipeline-pipeline

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: |
          pytest tests/ --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Init
        working-directory: terraform
        run: terraform init
      
      - name: Terraform Plan
        working-directory: terraform
        run: terraform plan
      
      - name: Terraform Apply
        working-directory: terraform
        run: terraform apply -auto-approve
        env:
          TF_VAR_db_username: ${{ secrets.DB_USERNAME }}
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
```

## Passo 6: Script de Deploy

**scripts/deploy.sh:**
```bash
#!/bin/bash

set -e

echo "🚀 Iniciando deploy..."

# Variáveis
ENVIRONMENT=${1:-dev}
AWS_REGION=${AWS_REGION:-us-east-1}
ECR_REPOSITORY="data-pipeline-pipeline"

# Build Docker
echo "📦 Construindo imagem Docker..."
docker build -t $ECR_REPOSITORY:latest .

# Login ECR
echo "🔐 Fazendo login no ECR..."
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY

# Push imagem
echo "⬆️  Fazendo push da imagem..."
docker tag $ECR_REPOSITORY:latest $ECR_REPOSITORY:latest
docker push $ECR_REPOSITORY:latest

# Deploy Terraform
echo "🏗️  Aplicando Terraform..."
cd terraform
terraform init
terraform plan -out=tfplan
terraform apply tfplan

echo "✅ Deploy concluído!"
```

## Passo 7: Makefile para Comandos Comuns

**Makefile:**
```makefile
.PHONY: help build up down test clean deploy

help:
	@echo "Comandos disponíveis:"
	@echo "  make build    - Construir imagens Docker"
	@echo "  make up       - Iniciar serviços"
	@echo "  make down     - Parar serviços"
	@echo "  make test     - Executar testes"
	@echo "  make clean    - Limpar recursos"
	@echo "  make deploy   - Deploy em produção"

build:
	docker-compose -f docker/docker-compose.yml build

up:
	docker-compose -f docker/docker-compose.yml up -d

down:
	docker-compose -f docker/docker-compose.yml down

test:
	docker-compose -f docker/docker-compose.yml run --rm pipeline pytest

clean:
	docker-compose -f docker/docker-compose.yml down -v
	docker system prune -f

deploy:
	./scripts/deploy.sh production

terraform-init:
	cd terraform && terraform init

terraform-plan:
	cd terraform && terraform plan

terraform-apply:
	cd terraform && terraform apply

terraform-destroy:
	cd terraform && terraform destroy
```

## Passo 8: Variáveis de Ambiente

**.env.example:**
```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=analytics_db

# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret

# Terraform
TF_VAR_db_username=postgres
TF_VAR_db_password=your_password
```

## Passo 9: Documentação

**README.md:**
```markdown
# Data Pipeline - Infrastructure as Code

## Pré-requisitos
- Docker
- Terraform
- AWS CLI configurado

## Desenvolvimento Local

```bash
# Iniciar ambiente local
make up

# Executar pipeline
docker-compose exec pipeline python main.py

# Ver logs
docker-compose logs -f pipeline
```

## Deploy

```bash
# Deploy em produção
make deploy

# Ou manualmente
./scripts/deploy.sh production
```

## Infraestrutura

A infraestrutura é gerenciada com Terraform:

```bash
cd terraform
terraform init
terraform plan
terraform apply
```
```

## Checklist de Conclusão

- [ ] Dockerfile criado
- [ ] Docker Compose configurado
- [ ] Terraform configurado
- [ ] CI/CD configurado
- [ ] Scripts de deploy criados
- [ ] Ambiente local funcionando
- [ ] Deploy em cloud testado
- [ ] Documentação completa
