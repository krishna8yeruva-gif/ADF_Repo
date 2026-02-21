# ADF_Repo

This repository is configured for GitHub integration with **Azure Data Factory (ADF)**.

## Repository Structure

| Folder | Description |
|---|---|
| `pipeline/` | ADF pipeline definitions (JSON) |
| `dataset/` | ADF dataset definitions (JSON) |
| `linkedService/` | ADF linked service definitions (JSON) |
| `trigger/` | ADF trigger definitions (JSON) |
| `dataflow/` | ADF data flow definitions (JSON) |
| `integrationRuntime/` | ADF integration runtime definitions (JSON) |

The `publish_config.json` file at the root tells ADF which branch to use when publishing ARM templates (defaults to `adf_publish`).

## Prerequisites — Authorize the AzureDataFactory GitHub OAuth App

> **Why AzureDataFactory doesn't appear in GitHub Settings yet:**
> The **AzureDataFactory** OAuth app is **not pre-installed**. It only appears in
> [GitHub → Settings → Authorized OAuth Apps](https://github.com/settings/connections/applications)
> *after* you have completed the authorization flow. The authorization is triggered
> from inside **ADF Studio** — you cannot add it directly from GitHub Settings.

To authorize the app, follow the full setup steps below. During step 5, GitHub will
automatically present an OAuth consent page asking you to grant **AzureDataFactory**
access to your repositories. Once you click **Authorize AzureDataFactory** on that
page, the app will appear in your GitHub Authorized OAuth Apps list and the
integration will proceed.

## Setting Up GitHub Integration in Azure Data Factory

Follow these steps to connect this repository to your ADF instance:

1. Open [Azure Data Factory Studio](https://adf.azure.com) and select your factory.
2. Click the **Manage** hub (toolbox icon) in the left navigation.
3. Under **Source control**, select **Git configuration**.
4. Click **Configure** and choose **GitHub** as the repository type.
5. ADF Studio will open a GitHub sign-in and authorization window (pop-up or redirect).
   Sign in with your GitHub account. GitHub will then show an OAuth consent page
   listing the permissions requested by **AzureDataFactory** — click
   **Authorize AzureDataFactory** to grant it access to your repositories.
   > After this step, **AzureDataFactory** will appear in
   > [GitHub → Settings → Authorized OAuth Apps](https://github.com/settings/connections/applications).
6. Back in ADF Studio, provide the following repository settings:
   - **GitHub Account**: `<your-github-account>` (e.g. `krishna8yeruva-gif`)
   - **Repository Name**: `<your-repository-name>` (e.g. `ADF_Repo`)
   - **Collaboration branch**: `main`
   - **Publish branch**: `adf_publish` (matches `publish_config.json`)
   - **Root folder**: `/`
7. Click **Apply**.

After connecting, ADF will read/write pipeline and resource definitions as JSON files into the folders above. When you click **Publish** in ADF Studio, ARM templates will be committed to the `adf_publish` branch automatically.

## Cost Reporting Agent

The `CostReportingAgent` pipeline queries two Azure REST APIs and assembles a
combined cost report for every ADF pipeline in your factory.

### How it works

| Step | Activity | What it does |
|---|---|---|
| 1 | `QueryPipelineRuns` | POSTs to the ADF Monitor API to list all pipeline runs in the report period |
| 2 | `QueryCostManagement` | POSTs to the Azure Cost Management API to get actual ADF spend grouped by resource and meter |
| 3 | `SetCostReport` | Merges both results into a single JSON report object |
| 4 | `IfWebhookConfigured` → `SendCostReport` | POSTs the report to your webhook URL (skipped if no URL is set) |

Authentication uses the Data Factory **Managed Identity** — no secrets or keys
are stored in the pipeline or this repository.

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
   **Add role assignment** → select **Cost Management Reader** → assign to the
   Managed Identity.
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
        "pipelineName": "IngestSalesData",
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

## CI/CD Notes

- Only the **development** ADF instance should be linked directly to this Git repository.
- Use the ARM templates from the `adf_publish` branch to deploy to **test** and **production** environments via your CI/CD pipeline (e.g., GitHub Actions or Azure DevOps).
