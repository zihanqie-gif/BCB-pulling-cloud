# BCB SGS Credit Benchmark Sync

This repository fetches Brazil Central Bank (BCB) SGS credit benchmark data and
upserts it into a Google Sheet. GitHub Actions runs the sync on a schedule or
when started manually.

## What the sync does

- Fetches monthly balance, nominal interest rate and NPL90 series for nine
  credit products (27 BCB SGS series in total).
- Converts balance from BRL millions to BRL billions.
- Pivots the series into one row per `reference_month` and `product`.
- Skips rows when any of the three metrics has not yet been published.
- Upserts by the key `(reference_month, product)` into the
  `raw_credit_benchmark` tab, updating existing rows and appending new ones.
- Requests the last four completed months each time it runs. Re-fetching this
  window allows recent BCB revisions to update existing rows.

The destination columns are:

| Column | Meaning |
| --- | --- |
| `reference_month` | Month in `YYYY-MM` format |
| `product` | Credit product name |
| `balance_brl_bn` | Balance in BRL billions |
| `nominal_rate_pa` | Nominal annual interest rate, as returned by BCB |
| `npl90` | NPL90 percentage, as returned by BCB |
| `updated_at` | UTC timestamp when the row was written |

## Requirements

- A Google account that can create or use a Google Cloud project and a Google
  spreadsheet.
- A GitHub account with permission to administer repository secrets and
  Actions.
- Python 3.12 to run the script locally (optional).

## 1. Create a Google Cloud project

1. Open the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a project, or select a dedicated project for this integration.
3. Note the project name so it is clear which project owns the service account.

## 2. Enable the Google Sheets API

1. In the selected project, open **APIs & Services → Library**.
2. Search for **Google Sheets API** and enable it.

This code opens the spreadsheet by ID using the Sheets API. It does not use
Google Drive search, so enabling the Drive API is not required for this setup.

## 3. Create a service account and JSON key

1. Open **IAM & Admin → Service Accounts** and create a service account, for
   example `bcb-sgs-sheet-writer`.
2. A project-level IAM role is not needed for writing to a spreadsheet that is
   explicitly shared with the service account. Avoid granting broad project
   roles for this purpose.
3. Open the service account, select **Keys → Add key → Create new key → JSON**.
4. Download the JSON key and keep it in a secure location. The `client_email`
   inside this file is the service account's Google identity.

Treat the JSON key as a password. Do not commit it, paste it into an issue, or
include it in screenshots. If a key is exposed, delete/revoke it in Cloud
Console and create a replacement.

## 4. Prepare the destination Google Sheet

1. Create or open the Google spreadsheet that should receive the data.
2. Copy its spreadsheet ID from the URL. It is the value between `/d/` and
   `/edit`, for example:

   ```text
   https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
   ```
   > The previous sheet ID was (if going to use the same): 1M-nRSJcN-8UhcW4THojN_XzhgSQf28YElBlAUw8ilgs/edit?gid=1704130958#gid=1704130958 

3. Share the spreadsheet with the service account's `client_email` from the
   downloaded JSON, and grant it **Editor** access.
4. The script writes to a tab named `raw_credit_benchmark`. It creates that tab
   and writes the headers if the tab does not exist. You can also create the tab
   yourself; the expected headers are the six columns listed above.

Sharing the spreadsheet with the service account is essential. Enabling an API
or creating the account alone does not grant access to the spreadsheet.

## 5. Add GitHub Actions secrets

In the GitHub repository, open **Settings → Secrets and variables → Actions**
and add these repository secrets:

| Secret name | Value |
| --- | --- |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | The complete contents of the downloaded service-account JSON key file |
| `SPREADSHEET_ID` | The spreadsheet ID copied in the previous step |

Paste the JSON file contents as-is; GitHub Secrets supports the multiline JSON
value. Do not add quotes around the entire JSON document. The workflow passes
both values to `scripts/bcb_sgs_sync.py` as environment variables.

If the repository uses GitHub Environments instead of repository-level secrets,
add the same names to the environment used by the workflow and configure the
job to use that environment.

## 6. Run and verify the workflow

1. In the repository, open **Actions → BCB SGS Credit Benchmark Sync**.
2. Choose **Run workflow** to start a manual run.
3. Open the run and inspect the **Run BCB SGS sync** step. A successful run
   reports how many rows were updated and appended.
4. Open the spreadsheet and confirm that `raw_credit_benchmark` has rows and
   the expected headers.

The scheduled workflow is in
[`.github/workflows/bcb_sgs_sync.yml`](.github/workflows/bcb_sgs_sync.yml). It
runs on the 5th, 15th and 25th of each month at 09:00 UTC (06:00 in
São Paulo). GitHub may delay scheduled runs during periods of high load. The
schedule runs from the repository's default branch; keep the workflow file
there. Use **Run workflow** when an immediate refresh is needed.

## Run locally (optional)

From the repository root, install the dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r scripts/requirements.txt
```

Set the environment variables from the key file and destination ID, then run
the script. Replace the example path and ID with your own values:

```bash
export GOOGLE_SERVICE_ACCOUNT_JSON="$(< /secure/path/service-account.json)"
export SPREADSHEET_ID="your_spreadsheet_id"
python scripts/bcb_sgs_sync.py
unset GOOGLE_SERVICE_ACCOUNT_JSON SPREADSHEET_ID
```

The script writes to the spreadsheet when it runs. Keep the key outside the
repository and remove the environment variable from your shell when finished.
The `.gitignore` excludes common local key files, but that is not a substitute
for keeping credentials out of the repository.

## Google Sheets versus Microsoft Excel

The existing integration writes to a **Google Sheet** through the Google
Sheets API. It does not connect to Microsoft Excel, OneDrive or SharePoint, and
it will not automatically populate a local `.xlsx` file.

If an Excel copy is sufficient, open the Google Sheet and use **File → Download
→ Microsoft Excel (.xlsx)** when you need a snapshot. That download is manual
and will not stay in sync. For a live Excel destination, the integration needs
to be changed to use Microsoft Graph and an Excel workbook stored in OneDrive
or SharePoint, with a Microsoft Entra app registration and its own credentials
and workbook permissions. Do not put Google service-account credentials into
Excel or reuse them for Microsoft Graph.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `KeyError: 'GOOGLE_SERVICE_ACCOUNT_JSON'` or `SPREADSHEET_ID` | Confirm both secrets are present under the repository or workflow environment used by the run. |
| Google API `403` or permission error | Confirm Google Sheets API is enabled and the spreadsheet is shared with the service account email as an Editor. |
| Spreadsheet not found | Recheck `SPREADSHEET_ID`; use only the ID, not the full URL. |
| JSON parse or credential error | Replace the secret with the complete, valid service-account JSON. If the key was exposed, revoke it and create a new one. |
| No rows written | Check the step logs for BCB/API errors and incomplete months. The script only writes months where all three metrics are available. |
| Scheduled run did not start at the expected minute | GitHub scheduled workflows can be delayed; check Actions and use the manual run option if needed. |

## Repository files

- [`scripts/bcb_sgs_sync.py`](scripts/bcb_sgs_sync.py): fetch, normalize, pivot
  and Google Sheets upsert logic.
- [`scripts/requirements.txt`](scripts/requirements.txt): Python dependencies.
- [`.github/workflows/bcb_sgs_sync.yml`](.github/workflows/bcb_sgs_sync.yml):
  scheduled and manual GitHub Actions workflow.
