# 🛵 Motoflow - Sistema de Gerenciamento de Motos

Sistema inteligente para gerenciamento e localização de motos em pátios de locadoras, desenvolvido em Spring Boot com tecnologia BLE.

## 🛠️ Tecnologias Utilizadas

| Componente | Tecnologia |
|------------|------------|
| Backend | Java 17 + Spring Boot 3.x |
| Banco de Dados | PostgreSQL 16 |
| Migração | Flyway |
| Segurança | Spring Security + JWT |
| Frontend | Thymeleaf + Bootstrap |
| Documentação API | Swagger/OpenAPI |
| Containerização | Docker |
| Cloud | Azure Container Registry + Azure Container Instances |

---

## 🚀 Pré-requisitos

- Java 17 ou superior
- Maven 3.6+
- Docker e Docker Compose
- Azure CLI (para deploy na nuvem)
- PostgreSQL 16+ (se executar sem Docker)

---

## 🏃‍♂️ Execução Local

### 1. Clone o repositório
```bash
git clone https://github.com/arthurspedine/motoflow-devops.git
cd motoflow-devops
```

### 2. Execução com Docker (Recomendado)

#### Banco de Dados PostgreSQL
```bash
# Build da imagem do PostgreSQL
docker build --platform=linux/amd64 -f Dockerfile.postgres -t motoflow-db:latest .

# Executar o container do banco
docker run -d \
  --name motoflow-postgres \
  -e POSTGRES_DB=motoflow-db \
  -e POSTGRES_USER=motoflow \
  -e POSTGRES_PASSWORD=motoflow \
  -p 5432:5432 \
  motoflow-db:latest
```

#### Aplicação Spring Boot
```bash
# Build da aplicação
docker build --platform=linux/amd64 -f Dockerfile -t motoflow-app:latest .

# Executar a aplicação
docker run -d \
  --name motoflow-app \
  -e DB_URL=jdbc:postgresql://host.docker.internal:5432/motoflow-db \
  -e DB_USERNAME=motoflow \
  -e DB_PASSWORD=motoflow \
  -e JWT_SECRET=meu-segredo-super-seguro-para-jwt-tokens \
  -p 8080:8080 \
  --link motoflow-postgres:postgres \
  motoflow-app:latest
```

#### Executar aplicação
```bash
# Compilar e executar
./mvnw clean install
./mvnw spring-boot:run

# Ou executar o JAR
java -jar target/motoflow-0.0.1-SNAPSHOT.jar
```

---

## ☁️ Deploy na Azure

### 1. Preparação do Ambiente
```bash
# Login no Azure
az login

# Criar Resource Group
az group create --name motoflow-rg --location eastus

# Criar Azure Container Registry
az acr create --resource-group motoflow-rg --name motoflow --sku Basic --admin-enabled true
```

### 2. Deploy do Banco de Dados
```bash
# Build e push da imagem do PostgreSQL
docker build --platform=linux/amd64 -f Dockerfile.postgres -t motoflow.azurecr.io/db:1.0 .
az acr login --name motoflow
docker push motoflow.azurecr.io/db:1.0

# Obter credenciais do ACR
az acr credential show --name motoflow --resource-group motoflow-rg --query username --output tsv
az acr credential show --name motoflow --resource-group motoflow-rg --query passwords --output tsv

# Deploy do container PostgreSQL
az container create \
  --resource-group motoflow-rg \
  --name motoflow-db \
  --image motoflow.azurecr.io/db:1.0 \
  --ports 5432 \
  --dns-name-label motoflow-db \
  --environment-variables POSTGRES_DB=motoflow-db POSTGRES_USER=motoflow POSTGRES_PASSWORD=motoflow \
  --cpu 1 --memory 2 \
  --restart-policy OnFailure \
  --os-type Linux

# Obter URL do banco
POSTGRES_FQDN=$(az container show --resource-group motoflow-rg --name motoflow-db --query ipAddress.fqdn --output tsv)
DATABASE_URL="jdbc:postgresql://$POSTGRES_FQDN:5432/motoflow-db"
echo "PostgreSQL disponível em: $POSTGRES_FQDN:5432"
```

### 3. Deploy da Aplicação
```bash
# Build e push da aplicação
docker build --platform=linux/amd64 -f Dockerfile -t motoflow.azurecr.io/motoflow-server:1.0 .
docker push motoflow.azurecr.io/motoflow-server:1.0

# Obter credenciais do ACR
az acr credential show --name motoflow --resource-group motoflow-rg --query username --output tsv
az acr credential show --name motoflow --resource-group motoflow-rg --query passwords --output tsv

# Deploy do container da aplicação
az container create \
  --resource-group motoflow-rg \
  --name motoflow-server \
  --image motoflow.azurecr.io/motoflow-server:1.0 \
  --ports 8080 \
  --dns-name-label motoflow-server \
  --environment-variables DB_URL=$DATABASE_URL DB_USERNAME=motoflow DB_PASSWORD=motoflow JWT_SECRET=meu-segredo-super-seguro \
  --cpu 1 --memory 2 \
  --restart-policy OnFailure \
  --os-type Linux

# Obter URL da aplicação
APP_URL=$(az container show --resource-group motoflow-rg --name motoflow-server --query ipAddress.fqdn --output tsv)
echo "Aplicação disponível em: http://$APP_URL:8080"
```

---

## 🧪 Testes da API

### Acesso à Documentação
- **Swagger UI**: `http://localhost:8080/swagger-ui.html`

### Scripts de Teste

#### 1. Autenticação
```bash
# Login (POST)
curl -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "8",
    "password": "Patio2025"
  }'
```

**Resposta esperada:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
}
```

#### 2. Criar Posição no Pátio
```bash
# Substituir {TOKEN} pelo token recebido no login
curl -X POST http://localhost:8080/api/posicoes/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {TOKEN}" \
  -d '{
    "setor": "A",
    "capacidadeSetor": 10
  }'
```

#### 3. Cadastrar Moto
```bash
curl -X POST http://localhost:8080/api/motos/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {TOKEN}" \
  -d '{
    "tipoMoto": "MOTTU_POP",
    "ano": 2022,
    "placa": "ABC6576",
    "precoAluguel": 150.00,
    "statusMoto": "DISPONIVEL",
    "setor": "A",
    "codRastreador": "1433ABC",
    "dataEntrada": "2025-09-14T22:30:00"
  }'
```

### Credenciais Padrão
- **Usuário**: `8`
- **Senha**: `Patio2025`

---

## 🗂️ Estrutura do Projeto

```
src/
├── main/
│   ├── java/br/com/fiap/motoflow/
│   │   ├── config/          # Configurações (Security, CORS, etc.)
│   │   ├── controller/      # Controllers REST e Web
│   │   ├── dto/             # Data Transfer Objects
│   │   ├── model/           # Entidades JPA
│   │   ├── repository/      # Repositórios Spring Data
│   │   ├── service/         # Regras de negócio
│   │   └── exceptions/      # Tratamento de exceções
│   └── resources/
│       ├── db/migration/    # Scripts Flyway
│       └── templates/       # Templates Thymeleaf
├── test/                    # Testes unitários
└── devops/                  # Scripts Docker e Azure
```

---

## 🧹 Limpeza do Ambiente

### Azure
```bash
# Remover resource group (remove todos os recursos)
az group delete --name motoflow-rg --yes --no-wait
```
