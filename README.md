# get-nuget-pat

GitHub composite action that fetches the Azure DevOps NuGet feed PAT from Azure Key Vault at CI time. Eliminates the need to store `AZURE_DEVOPS_PAT` as a per-repo GitHub Actions secret.

## Prerequisites

**`azure/login` must be called before this action.** This action requires an authenticated Azure CLI session — it does not handle Azure authentication itself. This keeps auth concerns in the caller (some workflows log in for other reasons too).

## Usage

```yaml
permissions:
  id-token: write   # required for OIDC login
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - uses: jasonberkes/get-nuget-pat@v1
        id: pat
        with:
          key-vault: tm-kv-prod-eus2

      - name: Authenticate NuGet feed
        run: |
          dotnet nuget update source tm-packages \
            --username az \
            --password "${{ steps.pat.outputs.value }}" \
            --store-password-in-clear-text \
            --configfile nuget.config
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `key-vault` | Yes | — | Azure Key Vault name (e.g. `tm-kv-prod-eus2`) |
| `secret-name` | No | `AzureDevOps--PAT` | Secret name in Key Vault |

## Outputs

| Name | Description |
|------|-------------|
| `value` | The PAT value. Automatically masked so it won't appear in logs. |

## Azure setup

The calling workflow's identity (Service Principal or managed identity) needs:

- **RBAC**: `Key Vault Secrets User` role on the Key Vault, or `Get` permission via access policy
- **OIDC federated credential** configured for your repo/branch if using OIDC login

For TaskMaster repos, use the `github-actions-taskmaster` Service Principal. The three `AZURE_*` values must be set as **repo-level Actions variables** (not secrets — they are non-sensitive IDs):

| Variable | Value |
|----------|-------|
| `AZURE_CLIENT_ID` | App ID of the Service Principal |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

## Troubleshooting

**`Login failed` / `AADSTS` error**: `azure/login` was not called before this action, or the SP doesn't have a federated credential matching your repo/branch.

**`SecretNotFound` / exit 1**: The secret name doesn't exist in the specified Key Vault, or the SP lacks `Key Vault Secrets User` RBAC role.

**HTTP 401 from NuGet feed**: The fetched PAT is valid but expired or the Key Vault secret holds a revoked value — rotate `AzureDevOps--PAT` in Key Vault.

**PAT appears unmasked in logs**: Stop immediately. Rotate `AzureDevOps--PAT` in Key Vault, then file a bug — the `::add-mask::` call should prevent this.

## Migration from per-repo AZURE_DEVOPS_PAT

1. Set `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` as repo-level Actions variables.
2. Add `permissions: id-token: write` to the job.
3. Add `azure/login@v2` step before NuGet auth.
4. Replace `--password "${{ secrets.AZURE_DEVOPS_PAT }}"` with `--password "${{ steps.pat.outputs.value }}"` using this action.
5. Open PR, verify CI green.
6. Merge and verify CI green on main.
7. Delete the repo's `AZURE_DEVOPS_PAT` secret: `gh secret delete AZURE_DEVOPS_PAT -R jasonberkes/<repo>`
8. Re-run CI on main to confirm deletion didn't break anything.
