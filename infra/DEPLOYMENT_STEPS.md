# Deploying db-service to Azure via GitHub Actions

Target architecture: **Azure Container Apps** (API) + **Azure Database for
PostgreSQL Flexible Server** (DB), provisioned with the Bicep template in
this folder and deployed by the workflows in `.github/workflows/`.

- `infra/main.bicep` — ACR, Log Analytics, Container Apps environment, Postgres Flexible Server (run rarely, only when infra changes)
- `.github/workflows/infra.yml` — provisions/updates the infra above (manual trigger)
- `.github/workflows/deploy.yml` — builds the image and deploys/updates the app (runs on every push to `main` that touches `db-service/`)

## 0. Prerequisites

- An Azure subscription, and the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) logged in (`az login`)
- This code pushed to a GitHub repository (it isn't a git repo yet):

  ```
  cd DevOps-Session
  git init
  git add .
  git commit -m "Initial commit"
  git branch -M main
  git remote add origin <your-github-repo-url>
  git push -u origin main
  ```

## 1. Create a resource group

```bash
az group create --name rg-dbservice --location eastus
```

## 2. Set up GitHub OIDC federation (no long-lived secrets)

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
RG=rg-dbservice
REPO=<github-org>/<github-repo>   # e.g. octocat/db-service

# App registration + service principal
APP_ID=$(az ad app create --display-name "db-service-github-oidc" --query appId -o tsv)
az ad sp create --id "$APP_ID"

# Federated credential trusting GitHub Actions on the main branch
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$REPO"':ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Also allow manual workflow_dispatch runs from any branch you trigger infra.yml from
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "github-main-dispatch",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$REPO"':ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Grant Contributor on the resource group
az role assignment create \
  --assignee "$APP_ID" \
  --role Contributor \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

echo "AZURE_CLIENT_ID=$APP_ID"
echo "AZURE_TENANT_ID=$TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID"
```

## 3. Add GitHub repo secrets and variables

In **Settings → Secrets and variables → Actions**:

**Secrets**
| Name | Value |
|---|---|
| `AZURE_CLIENT_ID` | from step 2 |
| `AZURE_TENANT_ID` | from step 2 |
| `AZURE_SUBSCRIPTION_ID` | from step 2 |
| `AZURE_RESOURCE_GROUP` | `rg-dbservice` |
| `POSTGRES_ADMIN_LOGIN` | e.g. `pgadmin` |
| `POSTGRES_ADMIN_PASSWORD` | a strong password |

**Variables** (added after step 4 below)
| Name | Value |
|---|---|
| `AZURE_LOCATION` | e.g. `eastus` |
| `APP_NAME` | e.g. `dbservice` |
| `ACR_NAME` | output of infra run |
| `CONTAINERAPP_ENV` | output of infra run |
| `POSTGRES_FQDN` | output of infra run |
| `CONTAINERAPP_NAME` | e.g. `dbservice-api` (optional, has a default) |

## 4. Provision infra

Run the **Provision Azure infra** workflow manually: GitHub repo → **Actions** → `infra.yml` → **Run workflow**.

It prints `ACR_NAME`, `CONTAINERAPP_ENV`, and `POSTGRES_FQDN` in the job log — copy those into the repo **Variables** listed above.

(Equivalent local command if you'd rather not use the workflow: `az deployment group create -g rg-dbservice --template-file infra/main.bicep --parameters postgresAdminLogin=<login> postgresAdminPassword=<password>`.)

## 5. Deploy the app

Push a change under `db-service/` to `main` (or run `deploy.yml` manually). It will:

1. Build the Docker image using `az acr build` (built in the cloud, no local Docker needed)
2. Create the Container App on first run, or update its image/secrets on subsequent runs

The job log prints the app's public FQDN at the end.

## 6. Verify

```bash
FQDN=<from the workflow log>
curl https://$FQDN/health
curl -X POST https://$FQDN/items -H "Content-Type: application/json" -d '{"name":"widget"}'
curl https://$FQDN/items
```

## 7. Tear down

```bash
az group delete --name rg-dbservice --yes --no-wait
```

Also remove the AD app registration if no longer needed:
```bash
az ad app delete --id "$APP_ID"
```

## Notes / production hardening

- ACR admin credentials are used in `deploy.yml` for simplicity. For production, disable `adminUserEnabled` in `main.bicep` and switch the Container App to pull via its system-assigned managed identity + an `AcrPull` role assignment.
- Postgres allows traffic from "any Azure service" (`0.0.0.0`/`0.0.0.0` firewall rule) because Container Apps Consumption plan has dynamic outbound IPs. For production, use VNet integration on both the Container Apps environment and Postgres instead.
- `infra.yml` is safe to re-run; Bicep deployments are idempotent.
