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

Before you can connect ADF to GitHub, you **must** authorize the **AzureDataFactory** OAuth app in GitHub to allow ADF to access your repositories.

1. Go to **[GitHub → Settings → Applications → Authorized OAuth Apps](https://github.com/settings/connections/applications)**.
2. If **AzureDataFactory** is already listed, click it and verify it has access to this repository (or all repositories).
3. If **AzureDataFactory** is **not** listed yet, it will appear for authorization the first time you attempt to link GitHub inside ADF Studio (step 5 below). GitHub will show an authorization prompt — click **Authorize AzureDataFactory** to grant access.

> **Note:** Without this authorization, the ADF portal will display an error or fail to list your repositories when you try to configure GitHub integration.

## Setting Up GitHub Integration in Azure Data Factory

Follow these steps to connect this repository to your ADF instance:

1. Open [Azure Data Factory Studio](https://adf.azure.com) and select your factory.
2. Click the **Manage** hub (toolbox icon) in the left navigation.
3. Under **Source control**, select **Git configuration**.
4. Click **Configure** and choose **GitHub** as the repository type.
5. When prompted, sign in to GitHub. If the **AzureDataFactory** OAuth app has not been authorized yet, GitHub will ask you to **Authorize AzureDataFactory** — click **Authorize** to allow ADF to access your repositories on your behalf.
6. Provide the following settings:
   - **GitHub Account**: `<your-github-account>` (e.g. `krishna8yeruva-gif`)
   - **Repository Name**: `<your-repository-name>` (e.g. `ADF_Repo`)
   - **Collaboration branch**: `main`
   - **Publish branch**: `adf_publish` (matches `publish_config.json`)
   - **Root folder**: `/`
7. Click **Apply**.

After connecting, ADF will read/write pipeline and resource definitions as JSON files into the folders above. When you click **Publish** in ADF Studio, ARM templates will be committed to the `adf_publish` branch automatically.

## CI/CD Notes

- Only the **development** ADF instance should be linked directly to this Git repository.
- Use the ARM templates from the `adf_publish` branch to deploy to **test** and **production** environments via your CI/CD pipeline (e.g., GitHub Actions or Azure DevOps).
