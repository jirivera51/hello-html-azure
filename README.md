# Deploy static HTML to Azure Web App (josehtmldemo)

This README provides clear, step-by-step instructions to deploy a simple static HTML site to an **Azure Web App** using **GitHub Actions (OIDC)**. This is for static sites (HTML/CSS/JS). If you’re working with ASP.NET or .NET projects, you will need additional build steps.

---

## Prerequisites
- **Azure Subscription** with permissions to create resources and role assignments.
- **Azure Active Directory tenant** where you can create an App Registration.
- **GitHub repository** with your static site (must contain `index.html`).
- Tools (optional but useful):
  - [Git](https://git-scm.com/)
  - [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
  - [Visual Studio Code](https://code.visualstudio.com/)

---

## Step 1 — Create Azure Resources

Using Azure CLI:
```bash
# Login to Azure
az login

# Show current subscription
az account show --output table

# Create a Resource Group
az group create --name myResourceGroup --location eastus

# Create App Service Plan
az appservice plan create --name myAppPlan --resource-group myResourceGroup --sku B1 --is-linux

# Create Web App (app name must be unique)
az webapp create --resource-group myResourceGroup --plan myAppPlan --name josehtmldemo --runtime "NODE|18-lts"

# Get the default hostname
az webapp show --name josehtmldemo --resource-group myResourceGroup --query defaultHostName -o tsv
```

---

## Step 2 — Azure AD App Registration (OIDC)
1. In Azure Portal: **Azure Active Directory → App registrations → New registration**.
   - Name: `github-oidc-josehtmldemo`
   - Note **Application (client) ID** and **Directory (tenant) ID**.

2. Assign **Contributor** role to the service principal on your Web App (or resource group).

3. In App Registration → **Certificates & secrets → Federated credentials → Add credential**:
   - Issuer: GitHub
   - Subject: `repo:<OWNER>/<REPO>:ref:refs/heads/master` (or `main`)
   - Audience: `api://AzureADTokenExchange`

---

## Step 3 — Add GitHub Secrets
In your GitHub repository → **Settings → Secrets and variables → Actions** add:
- `AZURE_CLIENT_ID` = App Registration’s Client ID
- `AZURE_TENANT_ID` = Directory (Tenant) ID
- `AZURE_SUBSCRIPTION_ID` = Azure Subscription ID

---

## Step 4 — GitHub Actions Workflow
Create `.github/workflows/deploy-to-azure.yml` in your repo:

```yaml
name: Deploy HTML to Azure Web App - josehtmldemo

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'josehtmldemo'
          package: '.'
```

> If your HTML is inside `wwwroot/` or another folder, change `package: '.'` to `package: './wwwroot'`.

---

## Step 5 — Validate Deployment
- Visit: `https://josehtmldemo.azurewebsites.net`
- Or run:
```bash
az webapp browse -n josehtmldemo -g myResourceGroup
```

---

## Troubleshooting
- **NuGet/MSBuild errors:** Remove all `nuget restore` or `msbuild` steps — not needed for static HTML.
- **OIDC token errors:** Ensure `permissions: id-token: write` is in the workflow and federated credentials match your repo/branch.
- **Permission denied:** Confirm the service principal has the correct role assigned to the Web App.

---

## Notes
- For static-only sites, **Azure Static Web Apps** may be easier.
- This guide uses OIDC (no client secret required).
- Workflow uses `ubuntu-latest` (faster, simpler for static sites).