# Go-Live Checklist - AIforITOps (GitHub Actions + Bicep + OIDC)

Persistent tracker for taking the deployment live. Safe to use across multiple chat windows - update the checkboxes as you complete each step. Commands assume `az` and `gh` CLIs are logged in.

Repo: `Dhana10/AIforITOps` | Branch: `main`

---

## 0. Set shared variables (run once per shell session)

```bash
APP_NAME="github-aiforitops-deploy"
GITHUB_REPO="Dhana10/AIforITOps"
SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
TENANT_ID="$(az account show --query tenantId -o tsv)"

# Deployment config
AZURE_ENV_NAME="aiops-prod"
AZURE_LOCATION="eastus"
AZURE_OPENAI_LOCATION="westus"
```

---

## 1. Set Azure subscription context

- [x] Confirm the correct subscription is active.

```bash
az account show --query "{name:name, id:id}" -o table
# If wrong: az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
```

---

## 2. Create Entra app registration + service principal

- [x] Create the app and SP, capture `APP_ID` and `SP_OBJECT_ID`.

```bash
az ad app create --display-name "$APP_NAME"
APP_ID="$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)"

az ad sp create --id "$APP_ID"
SP_OBJECT_ID="$(az ad sp show --id "$APP_ID" --query id -o tsv)"

echo "APP_ID=$APP_ID"
echo "SP_OBJECT_ID=$SP_OBJECT_ID"
```

---

## 3. Create federated credentials (OIDC)

- [x] Add credential for pushes to `main`.
- [x] (Optional) Add credential for the `production` environment / manual runs.

```bash
# main branch (push trigger)
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_REPO"':ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# optional: production environment (manual dispatch / approvals)
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "github-production",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:'"$GITHUB_REPO"':environment:production",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

---

## 4. Assign subscription roles

- [x] Contributor (create resources).
- [x] User Access Administrator (Key Vault RBAC role assignments in Bicep).

```bash
az role assignment create --assignee "$APP_ID" --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

az role assignment create --assignee "$APP_ID" --role "User Access Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

---

## 5. Set GitHub repository variables

- [x] Push all 7 variables via `gh`.

```bash
gh variable set AZURE_CLIENT_ID --repo "$GITHUB_REPO" --body "$APP_ID"
gh variable set AZURE_TENANT_ID --repo "$GITHUB_REPO" --body "$TENANT_ID"
gh variable set AZURE_SUBSCRIPTION_ID --repo "$GITHUB_REPO" --body "$SUBSCRIPTION_ID"
gh variable set AZURE_PRINCIPAL_ID --repo "$GITHUB_REPO" --body "$SP_OBJECT_ID"
gh variable set AZURE_ENV_NAME --repo "$GITHUB_REPO" --body "$AZURE_ENV_NAME"
gh variable set AZURE_LOCATION --repo "$GITHUB_REPO" --body "$AZURE_LOCATION"
gh variable set AZURE_OPENAI_LOCATION --repo "$GITHUB_REPO" --body "$AZURE_OPENAI_LOCATION"

gh variable list --repo "$GITHUB_REPO"
```

---

## 6. Verify Azure OpenAI gpt-4o quota

- [x] Confirm capacity for `gpt-4o` in `$AZURE_OPENAI_LOCATION` (default `westus`).

```bash
az cognitiveservices usage list --location "$AZURE_OPENAI_LOCATION" \
  --query "[?contains(name.value, 'OpenAI.Standard.gpt-4o')]" -o table
```

If no quota: pick another region, then update the variable:
`gh variable set AZURE_OPENAI_LOCATION --repo "$GITHUB_REPO" --body "<region>"`

---

## 7. Commit and push deployment files

- [x] Push `.github/workflows/deploy.yml` and `plan.md` to `main` (this triggers the workflow).

```bash
git add .github/workflows/deploy.yml plan.md GO-LIVE.md
git commit -m "Add GitHub Actions OIDC deployment workflow"
git push origin main
```

---

## 8. Trigger and monitor the workflow

- [x] The push above auto-triggers it. Or run manually.
- [x] Watch the run to completion.

```bash
# manual trigger (optional)
gh workflow run "Deploy to Azure" --repo "$GITHUB_REPO" --ref main

# watch latest run
gh run watch --repo "$GITHUB_REPO" "$(gh run list --repo "$GITHUB_REPO" --limit 1 --json databaseId -q '.[0].databaseId')"
```

---

## 9. Verify the deployment

- [x] Pods running, services have external IPs, apps reachable.

```bash
RESOURCE_GROUP="rg-$AZURE_ENV_NAME"
AKS_CLUSTER_NAME="$(az aks list -g "$RESOURCE_GROUP" --query '[0].name' -o tsv)"

az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_CLUSTER_NAME" --overwrite-existing
kubectl get pods -n ai-demo
kubectl get svc -n ai-demo
```

App URLs are also written to the workflow's **job summary** (StoreFront + AdminSite).

---

## Quick status

| # | Task | Done |
|---|------|------|
| 1 | Subscription context | [x] |
| 2 | Entra app + SP | [x] |
| 3 | Federated credentials | [x] |
| 4 | Subscription roles | [x] |
| 5 | GitHub variables | [x] |
| 6 | OpenAI quota | [x] |
| 7 | Push deployment files | [x] |
| 8 | Run workflow | [x] |
| 9 | Verify live | [x] |
