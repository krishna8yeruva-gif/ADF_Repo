# ADF_Repo

Azure Data Factory integration repository. This repo stores ADF pipeline definitions, linked services, datasets, triggers, and infrastructure as code managed through Git integration.

## Repository Structure

```
ADF_Repo/
├── pipeline/                  # ADF pipeline definitions
│   └── BlobToSqlPipeline.json # Copies CSV files from Blob Storage to SQL Database
├── linkedService/             # Connection definitions
│   ├── AzureBlobStorage.json  # Azure Blob Storage connection (via Key Vault secret)
│   ├── AzureKeyVault.json     # Azure Key Vault connection
│   └── AzureSqlDatabase.json  # Azure SQL Database connection (via Key Vault secret)
├── dataset/                   # Dataset definitions
│   ├── BlobInputDataset.json  # Parameterised CSV input from Blob Storage
│   └── SqlOutputDataset.json  # Parameterised SQL table output
├── trigger/                   # Trigger definitions
│   └── DailyTrigger.json      # Daily schedule trigger (midnight UTC)
├── integrationRuntime/        # Integration runtime definitions
├── factory/                   # ARM templates for ADF infrastructure
│   ├── ARMTemplateForFactory.json            # Full ARM template
│   └── ARMTemplateParametersForFactory.json  # ARM template parameters
└── .github/workflows/
    └── deploy-adf.yml         # CI/CD workflow for ADF deployment
```

## Getting Started

### Prerequisites

- An Azure subscription
- An Azure Resource Group
- An Azure Key Vault with the following secrets:
  - `AzureBlobStorageConnectionString` – connection string for the Blob Storage account
  - `AzureSqlConnectionString` – connection string for the Azure SQL Database

### Connecting ADF to this Repository

1. Open your Azure Data Factory instance in the Azure portal.
2. Navigate to **Manage → Git configuration**.
3. Select **GitHub** as the repository type and connect with your credentials.
4. Set the **Repository name** to `ADF_Repo`, the **Collaboration branch** to `main`, and the **Root folder** to `/`.
5. Click **Apply**.

### Deploying via GitHub Actions

The workflow in `.github/workflows/deploy-adf.yml` deploys changes to ADF automatically on every push to `main` that modifies ADF artefacts. It can also be triggered manually.

#### Required GitHub Secrets

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | Client ID of the Azure service principal (federated credential) |
| `AZURE_TENANT_ID` | Azure Active Directory tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `KEY_VAULT_BASE_URL` | Base URL of the Azure Key Vault (e.g. `https://my-kv.vault.azure.net/`) |

#### Required GitHub Variables

| Variable | Description |
|---|---|
| `AZURE_RESOURCE_GROUP` | Name of the Azure Resource Group |
| `ADF_FACTORY_NAME` | Name of the Azure Data Factory instance |

### Deploying Manually with Azure CLI

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file factory/ARMTemplateForFactory.json \
  --parameters factoryName=<your-adf-name> \
               keyVaultBaseUrl=https://<your-keyvault-name>.vault.azure.net/
```

## Pipeline Overview

### BlobToSqlPipeline

Copies delimited (CSV) files from Azure Blob Storage into an Azure SQL Database table using an upsert strategy.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `sourceFolder` | string | `data` | Blob container folder to read from |
| `targetSchema` | string | `dbo` | SQL schema name |
| `targetTable` | string | *(required)* | Destination SQL table name |
| `upsertKeys` | array | `[]` | Column names used as upsert keys |