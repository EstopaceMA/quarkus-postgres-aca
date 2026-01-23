# Azure Container Apps - Java Todo API

A Quarkus-based Todo API deployed to Azure Container Apps using Terraform and Azure Developer CLI (azd).

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        Azure Resource Group                    │
│                                                                │
│  ┌──────────────────┐    ┌──────────────────────────────────┐  │
│  │ Container        │    │ Container Apps Environment       │  │
│  │ Registry (ACR)   │───▶│                                  │  │
│  └──────────────────┘    │  ┌─────────────────────────────┐ │  │
│                          │  │ Backend Container App       │ │  │
│                          │  │ (Quarkus Todo API)          │ │  │
│                          |  |                             | |  |
│                          │  └─────────────────────────────┘ │  │
│                          └──────────────────────────────────┘  │
│                                         │                      │
│                                         ▼                      │
│                          ┌──────────────────────────────────┐  │
│                          │ PostgreSQL Flexible Server       │  │
│                          │ - Database: todo                 │  │
│                          └──────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Log Analytics Workspace                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
- [Terraform](https://www.terraform.io/downloads) >= 1.0
- [Docker](https://www.docker.com/get-started)
- [Java 21](https://adoptium.net/) (for local development)
- [Maven](https://maven.apache.org/) (for local development)

## Project Structure

```
.
├── azure.yaml                 # Azure Developer CLI configuration
├── infra/                     # Terraform infrastructure
│   ├── main.tf               # Main configuration using modules
│   ├── variables.tf          # Input variables
│   ├── outputs.tf            # Output values
│   ├── main.tfvars.json      # azd parameter file
│   └── modules/              # Terraform modules
│       ├── resource-group/   # Resource group module
│       ├── container-registry/# ACR module
│       ├── postgresql/       # PostgreSQL module
│       └── container-apps/   # Container Apps module
└── src/
    └── backend/              # Quarkus application
        ├── Dockerfile
        ├── pom.xml
        └── src/
            └── main/
                ├── java/     # Java source code
                └── resources/# Application configuration
```

## Quick Start

### 1. Login to Azure

```bash
azd auth login
```

### 2. Build the Application Locally

```bash
cd src/backend
./mvnw package -DskipTests
cd ../..
```

### 3. Deploy to Azure

```bash
azd up
```

This command will:

1. Prompt for environment name and Azure location
2. Provision all Azure resources using Terraform
3. Build and push the Docker image to ACR
4. Deploy the application to Container Apps

### 4. Access the Application

After deployment, the API will be available at the URL shown in the output:

```
BACKEND_URI = "https://backend.<unique-id>.azurecontainerapps.io"
```

## API Endpoints

| Method | Endpoint   | Description       |
| ------ | ---------- | ----------------- |
| GET    | /api/todos | List all todos    |
| POST   | /api/todos | Create a new todo |

### Example Usage

```bash
# Get all todos
curl https://backend.<unique-id>.azurecontainerapps.io/api/todos

# Create a todo
curl -X POST https://backend.<unique-id>.azurecontainerapps.io/api/todos \
  -H "Content-Type: application/json" \
  -d '{"description": "Learn Azure", "details": "Deploy apps to ACA"}'
```

## Infrastructure Modules

### resource-group

Creates an Azure Resource Group to contain all resources.

### container-registry

Provisions Azure Container Registry (ACR) for storing Docker images.

### postgresql

Deploys Azure Database for PostgreSQL Flexible Server with:

- Auto-generated secure password
- Firewall rule for Azure services
- Database named `todo`

### container-apps

Creates the Container Apps environment and application with:

- Log Analytics workspace for monitoring
- Ingress configuration for external access
- Environment variables for database connection
- Auto-scaling (0-3 replicas)

## Local Development

### Run PostgreSQL locally

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=todo \
  -p 5432:5432 \
  postgres:16
```

### Run the application

```bash
cd src/backend
./mvnw quarkus:dev
```

The API will be available at `http://localhost:8080/api/todos`

## Environment Variables

| Variable                                  | Description                | Default                               |
| ----------------------------------------- | -------------------------- | ------------------------------------- |
| QUARKUS_DATASOURCE_JDBC_URL               | PostgreSQL connection URL  | jdbc:postgresql://localhost:5432/todo |
| QUARKUS_DATASOURCE_USERNAME               | Database username          | postgres                              |
| QUARKUS_DATASOURCE_PASSWORD               | Database password          | postgres                              |
| QUARKUS_HIBERNATE_ORM_DATABASE_GENERATION | Schema generation strategy | drop-and-create (dev) / update (prod) |

## Commands Reference

```bash
# Deploy everything
azd up

# Provision infrastructure only
azd provision

# Deploy application only (after code changes)
azd deploy

# View deployed resources
azd show

# Tear down all resources
azd down

# View logs
azd monitor --logs
```

## Troubleshooting

### Build fails with Java version error

Ensure you're using Java 21:

```bash
java -version
export JAVA_HOME="/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home"
```

### Container App fails to start

Check the logs:

```bash
az containerapp logs show \
  --name backend \
  --resource-group <resource-group> \
  --follow
```

### Database connection issues

Verify the PostgreSQL firewall allows Azure services:

```bash
az postgres flexible-server firewall-rule list \
  --resource-group <resource-group> \
  --name <server-name>
```

## Clean Up

To delete all Azure resources:

```bash
azd down
```

## License

MIT
