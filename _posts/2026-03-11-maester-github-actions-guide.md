# Continuous Microsoft 365 Security Monitoring with Maester and GitHub Actions

Modern Microsoft 365 environments change constantly. New policies are
added, configurations drift, and security settings evolve as teams
deploy new services. Without continuous monitoring, it's easy for
misconfigurations to creep in unnoticed.

**Maester** is an open‑source framework designed to test Microsoft 365
security configurations automatically. It runs a suite of checks against
your tenant and reports configuration issues across services such as
Entra ID, Exchange Online, Teams, and Azure.

One of the most powerful ways to use Maester is to run it automatically
with **GitHub Actions**, giving you scheduled, repeatable security
checks and a historical record of results.

In this guide we will cover:

-   Setting up Maester for automation
-   Creating the Entra application and permissions
-   Creating the GitHub Actions workflow
-   Setting the required GitHub organisation permissions
-   Ensuring repositories remain private
-   Retaining six months of security reports automatically

------------------------------------------------------------------------

# Setting Up Maester for Automation

Maester supports running through **GitHub Actions with workload identity
federation**, meaning GitHub can authenticate to Azure without storing
client secrets.

The architecture looks like this:

GitHub Actions → Entra App Registration → Microsoft Graph / Exchange /
Teams / Azure APIs → Maester Security Tests

The workflow will:

1.  Authenticate to Azure using GitHub OIDC
2.  Run Maester tests
3.  Store the results in the repository
4.  Retain six months of reports automatically

------------------------------------------------------------------------

# Creating the Entra Application

1.  Open **Microsoft Entra admin center**
2.  Navigate to **Identity → Applications → App registrations**
3.  Select **New registration**

Enter:

Name: Maester DevOps Account\
Supported account types: Single tenant

Select **Register**.

Make note of:

-   Application (client) ID
-   Directory (tenant) ID

These will later be added to GitHub as secrets.

------------------------------------------------------------------------

# Granting Microsoft Graph Permissions

Open the application:

API Permissions → Add Permission

Choose:

Microsoft Graph → Application permissions

Add the following permissions:

-   Directory.Read.All
-   DirectoryRecommendations.Read.All
-   Policy.Read.All
-   Policy.Read.ConditionalAccess
-   Reports.Read.All
-   ReportSettings.Read.All
-   RoleManagement.Read.All
-   RoleEligibilitySchedule.Read.Directory
-   PrivilegedAccess.Read.AzureAD
-   IdentityRiskEvent.Read.All
-   UserAuthenticationMethod.Read.All
-   DeviceManagementConfiguration.Read.All
-   DeviceManagementManagedDevices.Read.All
-   DeviceManagementRBAC.Read.All
-   SecurityIdentitiesHealth.Read.All
-   SecurityIdentitiesSensors.Read.All
-   ThreatHunting.Read.All
-   SharePointTenantSettings.Read.All
-   OnPremDirectorySynchronization.Read.All

Then select **Grant admin consent**.

Optional permission:

-   ReportSettings.ReadWrite.All

------------------------------------------------------------------------

# Granting Exchange Online Permissions

Add:

Office 365 Exchange Online → Application permissions →
Exchange.ManageAsApp

Grant admin consent.

Then run:

``` powershell
New-ServicePrincipal -AppId <Application ID> -ObjectId <Service Principal Object ID> -DisplayName "Maester DevOps Account"

New-ManagementRoleAssignment -Role "View-Only Configuration" -App "Maester DevOps Account"
```

## ⚠️ Common Issue: Exchange Object ID

If you see:

AADServicePrincipalNotFound

You are likely using the **App Registration Object ID instead of the
Enterprise Application Object ID**.

Use:

Enterprise Applications → Maester App → Object ID

------------------------------------------------------------------------

# Granting Microsoft Teams Permissions

1.  Go to **Roles and administrators**
2.  Search for **Teams Reader**
3.  Select **Add assignment**
4.  Select the Maester application
5.  Assign permanently

------------------------------------------------------------------------

# Granting Azure Permissions

Run:

``` powershell
$servicePrincipal = "<Enterprise Application Object ID>"

Install-Module Az.Accounts -Force
Install-Module Az.Resources -Force

Connect-AzAccount

Invoke-AzRestMethod -Path "/providers/Microsoft.Authorization/elevateAccess?api-version=2015-07-01" -Method POST

New-AzRoleAssignment -ObjectId $servicePrincipal -Scope "/" -RoleDefinitionName "Reader"

New-AzRoleAssignment -ObjectId $servicePrincipal -Scope "/providers/Microsoft.aadiam" -RoleDefinitionName "Reader"
```

This grants the application tenant‑wide read visibility.

------------------------------------------------------------------------

# Configure GitHub Federation

In the Entra App:

Certificates & secrets → Federated credentials → Add credential

Scenario:

GitHub Actions deploying Azure resources

Fill:

Organisation: your-org\
Repository: maester-security\
Entity type: Branch\
Branch: main\
Credential name: maester-github

Save.

------------------------------------------------------------------------

# Adding GitHub Secrets

In GitHub:

Settings → Secrets and variables → Actions

Add:

-   AZURE_TENANT_ID
-   AZURE_CLIENT_ID

------------------------------------------------------------------------

# Creating the GitHub Actions Workflow

Create:

.github/workflows/run-maester.yml

Example:

``` yaml
name: Run Maester 🔥

on:
  push:
    branches:
      - main
    paths-ignore:
      - "reports/**"

  schedule:
    - cron: "30 7 * * *"

  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write

    steps:
      - name: Checkout repo
        uses: actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Run Maester 🔥
        id: maester
        uses: maester365/maester-action@main
        with:
          tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          client_id: ${{ secrets.AZURE_CLIENT_ID }}
          include_public_tests: true
          include_private_tests: true
          include_exchange: true
          include_teams: true
          maester_version: latest
          disable_telemetry: false
          step_summary: true

      - name: Write status 📃
        shell: bash
        run: |
          echo "The result of the test run is: ${{ steps.maester.outputs.result }}"
          echo "Total tests: ${{ steps.maester.outputs.tests_total }}"
          echo "Passed tests: ${{ steps.maester.outputs.tests_passed }}"
          echo "Failed tests: ${{ steps.maester.outputs.tests_failed }}"
          echo "Skipped tests: ${{ steps.maester.outputs.tests_skipped }}"

      - name: Archive results into weekly folder
        shell: bash
        run: |
          set -euo pipefail

          YEAR="$(date -u +%G)"
          WEEK="$(date -u +%V)"
          RUNSTAMP="$(date -u +%Y-%m-%dT%H-%M-%SZ)"

          DEST="reports/${YEAR}-W${WEEK}/${RUNSTAMP}"
          mkdir -p "$DEST"

          if [ ! -d "test-results" ]; then
            echo "test-results folder not found"
            exit 1
          fi

          cp -R test-results/. "$DEST"/

          cat > "$DEST/run-summary.txt" <<EOF
          Run timestamp (UTC): ${RUNSTAMP}
          Result: ${{ steps.maester.outputs.result }}
          Total tests: ${{ steps.maester.outputs.tests_total }}
          Passed tests: ${{ steps.maester.outputs.tests_passed }}
          Failed tests: ${{ steps.maester.outputs.tests_failed }}
          Skipped tests: ${{ steps.maester.outputs.tests_skipped }}
          Workflow run: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
          Commit: ${{ github.sha }}
          EOF

      - name: Update latest pointer
        shell: bash
        run: |
          set -euo pipefail

          YEAR="$(date -u +%G)"
          WEEK="$(date -u +%V)"
          RUNSTAMP="$(date -u +%Y-%m-%dT%H-%M-%SZ)"
          DEST="reports/${YEAR}-W${WEEK}/${RUNSTAMP}"

          mkdir -p reports/latest
          find reports/latest -mindepth 1 -maxdepth 1 -exec rm -rf {} +
          cp -R "$DEST"/. reports/latest/

      - name: Prune report history to 26 weeks
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p reports

          mapfile -t WEEK_DIRS < <(
            find reports -mindepth 1 -maxdepth 1 -type d -name '20*-W*' -printf '%f\n' | sort
          )

          COUNT="${#WEEK_DIRS[@]}"
          KEEP=26

          if [ "$COUNT" -le "$KEEP" ]; then
            echo "Nothing to prune. Found $COUNT weekly folders."
            exit 0
          fi

          DELETE_COUNT=$((COUNT - KEEP))

          for ((i=0; i<DELETE_COUNT; i++)); do
            echo "Deleting reports/${WEEK_DIRS[$i]}"
            rm -rf "reports/${WEEK_DIRS[$i]}"
          done

      - name: Commit weekly reports
        shell: bash
        run: |
          set -euo pipefail

          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git add reports

          if git diff --cached --quiet; then
            echo "No report changes to commit."
            exit 0
          fi

          git commit -m "Add Maester reports [skip ci]"
          git push
```

------------------------------------------------------------------------

# Retaining 26 Weeks of Reports

Reports are stored like:

reports/YYYY-Www/`<timestamp>`{=html}/

Example:

reports/ latest/ 2026-W11/ 2026-03-11T07-30Z/ 2026-W12/
2026-03-18T07-30Z/

The workflow automatically deletes folders older than **26 weeks**.

------------------------------------------------------------------------

# Setting GitHub Organisation Permissions

Organisation Settings → Actions → General → Workflow permissions

Set:

Read and write permissions

------------------------------------------------------------------------

## ⚠️ Common Issue: contents: write

If report commits fail, check your workflow contains:

``` yaml
permissions:
  contents: write
```

If your organisation enforces **read‑only workflow tokens**, you must
request an admin to enable write access or use a GitHub App / PAT.

------------------------------------------------------------------------

# Ensuring the Repository is Private

Set:

Settings → General → Change repository visibility → Private

Security test reports should never be stored in public repositories.

------------------------------------------------------------------------

# Final Thoughts

Automating Maester with GitHub Actions provides a lightweight but
powerful security monitoring pipeline for Microsoft 365.

Benefits include:

-   Automated daily security validation
-   Historical configuration reporting
-   Git-based audit trail
-   No stored secrets (OIDC authentication)
-   Automatic report retention

For organisations managing Microsoft 365 environments, this approach
provides an effective way to implement **continuous security
configuration monitoring**.
