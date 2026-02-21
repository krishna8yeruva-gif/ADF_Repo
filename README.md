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

## Setting Up GitHub Integration in Azure Data Factory

Follow these steps to connect this repository to your ADF instance:

1. Open [Azure Data Factory Studio](https://adf.azure.com) and select your factory.
2. Click the **Manage** hub (toolbox icon) in the left navigation.
3. Under **Source control**, select **Git configuration**.
4. Click **Configure** and choose **GitHub** as the repository type.
5. Authenticate with GitHub and provide the following settings:
   - **GitHub Account**: `<your-github-account>` (e.g. `krishna8yeruva-gif`)
   - **Repository Name**: `<your-repository-name>` (e.g. `ADF_Repo`)
   - **Collaboration branch**: `main`
   - **Publish branch**: `adf_publish` (matches `publish_config.json`)
   - **Root folder**: `/`
6. Click **Apply**.

After connecting, ADF will read/write pipeline and resource definitions as JSON files into the folders above. When you click **Publish** in ADF Studio, ARM templates will be committed to the `adf_publish` branch automatically.

## CI/CD Notes

- Only the **development** ADF instance should be linked directly to this Git repository.
- Use the ARM templates from the `adf_publish` branch to deploy to **test** and **production** environments via your CI/CD pipeline (e.g., GitHub Actions or Azure DevOps).
