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

## Branches

| Branch | Purpose |
|---|---|
| `main` | Collaboration branch — ADF Studio authors pipelines here; source of truth for `venkatazdf` |
| `adf_publish` | Auto-generated publish branch — ADF Studio writes ARM templates here on every **Publish** |
| `ADF_Github_20260221` | Snapshot of `main` as of 2026-02-21; used for point-in-time reference / environment promotion |

### Creating `ADF_Github_20260221` from `main`

This branch is a copy of `main` as it stood on 2026-02-21. Use any of the following methods:

**Option 1 — GitHub UI (easiest)**
1. Go to [github.com/krishna8yeruva-gif/ADF_Repo](https://github.com/krishna8yeruva-gif/ADF_Repo)
2. Click the **branch selector** (`main ▾`) at the top-left of the file list
3. Type `ADF_Github_20260221` in the search box
4. Click **Create branch: ADF_Github_20260221 from 'main'**

**Option 2 — GitHub CLI**
```bash
gh api repos/krishna8yeruva-gif/ADF_Repo/git/refs \
  --method POST \
  --field ref="refs/heads/ADF_Github_20260221" \
  --field sha="$(gh api repos/krishna8yeruva-gif/ADF_Repo/git/ref/heads/main --jq '.object.sha')"
```

**Option 3 — Git command line**
```bash
git fetch origin main
git checkout -b ADF_Github_20260221 origin/main
git push origin ADF_Github_20260221
```

---

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

## Cost Reporting Agent

The `CostReportingAgent` pipeline queries the ADF Monitor API and Azure Cost Management
API to produce a per-pipeline cost and usage report for your factory.

### How it works

| Step | Activity | What it does |
|---|---|---|
| 1 | `QueryPipelineRuns` | POSTs to the ADF Monitor API to list all pipeline runs in the report period |
| 2 | `QueryCostManagement` | POSTs to the Azure Cost Management API to get actual ADF spend grouped by resource and meter |
| 3 | `SetCostReport` | Merges both results into a single JSON report object |
| 4 | `IfWebhookConfigured` → `SendCostReport` | POSTs the report to your webhook URL (skipped if no URL is set) |

Authentication uses the Data Factory **Managed Identity** — no secrets or keys
are stored in the pipeline or this repository.

### Files added by this feature

| File | Description |
|---|---|
| `pipeline/CostReportingAgent.json` | Four-activity ADF pipeline definition |
| `trigger/CostReportDailyTrigger.json` | Daily schedule trigger (00:00 UTC, previous day window) |

### Pipeline parameters

| Parameter | Description | Example |
|---|---|---|
| `subscriptionId` | Azure subscription that owns the factory | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `resourceGroupName` | Resource group of the factory | `my-rg` |
| `dataFactoryName` | Name of the Data Factory instance | `my-adf` |
| `reportStartDate` | ISO-8601 start of the reporting window | `2024-01-01T00:00:00Z` |
| `reportEndDate` | ISO-8601 end of the reporting window | `2024-01-31T23:59:59Z` |
| `notificationWebhookUrl` | URL to receive the report payload (optional) | `https://hooks.example.com/report` |

### Required RBAC roles

Assign these roles to the Data Factory **Managed Identity** before running the pipeline:

| Role | Scope | Reason |
|---|---|---|
| **Monitoring Reader** | Data Factory resource | Read pipeline run history via ADF Monitor API |
| **Cost Management Reader** | Subscription | Read cost data via Azure Cost Management API |

To assign roles in the Azure Portal:
1. Open the Data Factory → **Managed identity** — note the Object (principal) ID.
2. Go to **Subscriptions** → your subscription → **Access control (IAM)** →
   **Add role assignment** → select **Cost Management Reader** → assign to the Managed Identity.
3. Repeat at the Data Factory resource level for **Monitoring Reader**.

### Running the agent manually

1. In ADF Studio, open the **Author** hub → **Pipelines** → `CostReportingAgent`.
2. Click **Debug** or **Add trigger → Trigger now**.
3. Fill in the parameter values and click **OK**.
4. Monitor progress in the **Monitor** hub → **Pipeline runs**.

### Enabling the daily schedule

The `CostReportDailyTrigger` runs the agent every day at **00:00 UTC**, reporting
on the previous day.

1. Open `trigger/CostReportDailyTrigger.json` and replace the placeholders:
   - `<your-subscription-id>`
   - `<your-resource-group>`
   - `<your-data-factory-name>`
   - `<your-webhook-url>` (or leave empty to skip delivery)
2. Publish the changes from ADF Studio.
3. In ADF Studio, go to **Manage** → **Triggers** → select `CostReportDailyTrigger`
   → click **Activate**.

### Webhook payload shape

When `notificationWebhookUrl` is set, the pipeline POSTs a JSON body like this:

```json
{
  "reportGeneratedAt": "2024-01-02T00:00:05Z",
  "reportPeriod": {
    "start": "2024-01-01T00:00:00Z",
    "end":   "2024-01-02T00:00:00Z"
  },
  "factory": "my-adf",
  "pipelineRuns": {
    "value": [
      {
        "pipelineName": "BlobToSqlPipeline",
        "runId": "...",
        "status": "Succeeded",
        "durationInMs": 43210,
        "runStart": "2024-01-01T08:00:00Z",
        "runEnd":   "2024-01-01T08:00:43Z"
      }
    ]
  },
  "costData": {
    "properties": {
      "columns": [
        { "name": "Cost",          "type": "Number" },
        { "name": "ResourceId",    "type": "String" },
        { "name": "MeterCategory", "type": "String" }
      ],
      "rows": [
        [ 1.23, "/subscriptions/.../factories/my-adf", "Azure Data Factory v2" ]
      ]
    }
  }
}
```
