# Risk_Automate_Md
## Overview
This document describes the implemented Extract → analysis flow for Self-Reporting, GPS, Tech Blocklist, and Product Misuse. **Voice is intentionally
excluded** until the table can be built from the **1-Many Dashboard** warehouse raw table/query is built. The idea is to automate the risk flow of how analysis is been done within the risk
NOTE: Most of the script which was written was derived from an existing template to follow. 

## Architecture at a glance
<img width="701" height="929" alt="image (2)" src="https://github.com/user-attachments/assets/def0674e-7cd2-485c-82c8-f46910466796" />

```text
Tiko data warehouse
        │
        │  python -m extract.run_extract --month YYYY-MM
        │  SQL in extract/sql/
        ▼
Monthly Extract files
data/<COUNTRY>/<YYYY-MM>/extract/
        │
        ├── Self Reporting ── dedicated Self Reporting SQL ──► workbook / Drive
        ├── GPS ────────────── Rafiki Journey + Tiko Miles ───► GPS workbook
        ├── Tech Blocklist ─── auto-blocklist + Match_Count
        │                     + Tiko Miles ───────────────────► workbook / Drive
        └── Product Misuse ─── DataBase Data ─────────────────► workbook / Drive
```
The **Extract** layer owns warehouse SQL and writes monthly source files. The tool scripts read those files, apply tool-specific analysis rules, and produce workbooks and (where configured) Drive outputs. Run the Extract for a month before running an analysis that depends on its files.


## Getting started

Use Python 3.12 (the version used by GitHub Actions) and run these commands
from the repository root in PowerShell. The dependencies are declared in
`requirements.txt`; install from that file instead of installing packages
individually:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Before running a market locally, ensure the repository's `.env` or an `ENV_FILE` contains that market's warehouse and Google Drive/authentication settings.
$env:PYTHONUTF8 = "1"
$month = "2026-09"
$country = "ZA"

## What the Extract produces
Run the full monthly warehouse extraction:

```powershell
python -m extract.run_extract --month 2026-09 --country ZA
```

Use `--no-upload` to write local Extract files only:

```powershell
python -m extract.run_extract --month 2026-09 --country ZA --no-upload
```

The standard Extract runs:

| SQL file | Extract output | Used by |
|---|---|---|
| `extract/sql/rafiki_journey.sql` | `Raw_Data__Rafiki_Journey_Raw_Data.csv` | GPS |
| `extract/sql/database_data.sql` | `Raw_Data__DataBase_Data.csv` | Product Misuse |
| `extract/sql/tiko_incentives.sql` | `Raw_Data__Tiko_Miles.csv` | GPS and Tech Blocklist |
| derived from `database_data.sql` | `Raw_Data__Match_Count.csv`, `Raw_Data__raw_Auto_blocklist.csv` | Tech Blocklist |

Files are saved beneath `data/<COUNTRY>/<YYYY-MM>/extract/` and the Extract workbook is published under the configured Drive Extract path. The standard SQL uses the selected month and the configured country/timezone. For example,with the default South Africa label, September files begin `SA_SEPT_2026_Raw_Data__`.

The Extract runs Self Reporting by default when its SQL files are present. It can also be run independently without rerunning the other Extract queries:
```powershell
python -m extract.run_extract --month 2026-09 --self-reporting-only --country ZA
python -m extract.run_extract --month 2026-09 --self-reporting-only --no-upload
```


## Self Reporting

Self Reporting has its own SQL queries under `extract/sql/`:

| SQL file | Purpose |
|---|---|
| `self_reporting_raw.sql` | Raw self-report fields, including `Day of Report Occured At` and `report_date` |
| `self_reporting_mvc.sql` | Raw MVC/Rafiki service validations |
| `self_reporting_mvc_aggregated.sql` | Facility/service comparison of report values and MVC Uptakes |
| `self_reporting_facilities.sql` | Facility list used for missing-facility analysis |


The occurrence timestamp is used for the selected reporting-month window; `report_date` is retained for review, not as the period filter. The comparison combines age-banded report fields with their base service type and removes identical source rows. Offboarded ProNursing Katlehong is excluded. The  Unverified rule is `1` when `report_field_value > MVC Uptakes`, otherwise `0`; Financial Impact is `Unverified × Cost`.

After extracting Self Reporting, build its analysis workbook:

```powershell
python -m src.self_reporting.run_self_reporting --month 2026-09 --country ZA
```
Add `--no-upload` to keep the workbook local. Outputs are saved under `data/<COUNTRY>/<YYYY-MM>/self_reporting/` and published under the configured Self Reporting Drive path.



## GPS

GPS does **not** have a separate GPS warehouse query. It uses the monthly `Rafiki_Journey_Raw_Data.csv` and `Tiko_Miles.csv` produced by the standard Extract. The analysis compares facility and offer coordinates using the
Haversine distance. Its default deduction rule is distance **greater than 500 m** with geolocation accuracy **at most 100 m**.

```powershell
python -m src.gps.run_gps_summary --month 2026-09 --country ZA
```
The command reads both source files from that month’s local Extract folder and writes a workbook under `data/<COUNTRY>/<YYYY-MM>/gps/`. To use custom CSVs, provide the raw file positionally and optionally a Tiko Miles file:

```powershell
python -m src.gps.run_gps_summary "C:\path\journey.csv" --tiko-miles "C:\path\tiko_miles.csv" --month 2026-09 --country ZA
```

If a Google Sheet should receive the results, provide its ID with `--sheet-id`. The default run writes a local workbook; it does not create a Drive output folder or upload automatically.

## Tech Blocklist

Tech Blocklist has no separate warehouse SQL. It consumes three files created by the standard Extract:

* `Raw_Data__raw_Auto_blocklist.csv` as `Raw`
* `Raw_Data__Match_Count.csv` as `Pivot Table 5`
* `Raw_Data__Tiko_Miles.csv` as `Tiko_Miles`

The analysis builds suspension and deduction tabs from these sources. The suspension threshold is more than 20 pre-merge profiles; the deduction rule uses a merge count of at least 5.

```powershell
python -X utf8 -m src.tech_blocklist.TechBlocklist --month 2026-09 --country ZA
```

This writes the workbook locally under
`data/<COUNTRY>/<YYYY-MM>/tech_blocklist/`. Add `--upload` to publish it to the
configured Tech Blocklist Drive path:

```powershell
python -m src.tech_blocklist.TechBlocklist --month 2026-09 --country KE --upload
```

## Product Misuse

Product Misuse reads the selected month’s `Raw_Data__DataBase_Data.csv` file. That file is produced by `extract/sql/database_data.sql` as part of the standard Extract. The analysis applies the configured product watchlist and minimum-event threshold per Rafiki within the selected month. The default threshold is three matching events (`MIN_OCCURRENCES` can override it). If no Rafiki reaches the threshold, the flagged-event tabs contain headers but no data; run metadata still records how many source rows and watchlist events were processed. **THIS SCRIPT WAS PRODUCED BY Kayode**

```powershell
python -m extract.run_extract --month 2026-09 --country ZA
```

```powershell
python -m src.product_misuse.product_misuse --month 2026-09 --country ZA
```

This writes the workbook locally under `data/<COUNTRY>/<YYYY-MM>/tech_blocklist/`. Add `--upload` to publish it to the configured Tech Blocklist Drive path:

```powershell
python -m src.product_misuse.product_misuse --month 2026-09 --country ZA --no-upload
```

The command writes a workbook to `data/<COUNTRY>/<YYYY-MM>/product_misuse/` and publishes the results in the configured output format under `src/Product Misuse/<COUNTRY>/<YYYY-MM>` (the default output format is a Google Sheet). Use `--no-upload` for a local-only run. To analyze a particular DataBase Data CSV instead of the selected month’s default file:

```powershell
python -m src.product_misuse.product_misuse --month 2026-09  --country ZA  --input "C:\path\DataBase_Data.csv" --no-upload
```
If the monthly input is missing, create it first:

```powershell
python -m extract.run_extract --month 2026-09
```

### Running different markets independently

Pass the market country code directly with `--country` to every completed
tool. Country codes are normalized for the warehouse, and output naming uses the country
label (`ZA` maps to `SA`; `KE` maps to `KE`) so runs don't inherit a different
market label from the default `.env`:

```powershell
$month = "2026-09"
$country = "KE" # or "ZA"

python -m extract.run_extract --month $month --country $country
python -m src.self_reporting.run_self_reporting --month $month --country $country
python -m src.gps.run_gps_summary --month $month --country $country
python -m src.tech_blocklist.TechBlocklist --month $month --country $country
python -m src.product_misuse.product_misuse --month $month --country $country
```

Each user can run a different country in their own PowerShell session at the same time, using the same shared code and warehouse credentials. Extract files and Drive paths are separated by country label and month as long as custom `DRIVE_*_PATH` settings retain the `{COUNTRY}` and `{YYYY}-{MM}` placeholders. For market-specific configuration beyond the country code, use a market-specific environment file. This keeps additional settings isolated when several users run the tools simultaneously.


## Month selection and configuration

Use `--month YYYY-MM` to select the reporting month. The shared `Period` settings control country, file names, local data folders, and Drive paths. When the command supports the shared default and `--month` is omitted, it uses `ANALYSIS_MONTH` from the environment or the previous calendar month.

Drive publishing also depends on the repository’s local environment configuration and Google authentication. Keep credentials in local secrets or environment variables; do not add them to this documentation or commit them. Relevant settings include the Drive root/path, output format, country, and Google authentication mode. See `.env` for setting names and safe placeholders.

## Voice
Voice is not part of this workflow yet. Its warehouse raw table/query has not been built, so this overview does not document or run a Voice Extract or analysis step. 


## Running markets in parallel with GitHub Actions

To start the market workflow:

1. In `.github/markets.yml`, set `enabled: true` only for markets ready to run. Confirm each market's warehouse country code, unique output label, access, and Drive configuration first.
2. In GitHub, open **Settings → Secrets and variables → Actions** and configure the `TIKO_DW_DSN` and `GOOGLE_SA_JSON` repository secrets and the `DRIVE_ROOT_FOLDER_ID` repository variable.
3. Commit and push the workflow and market configuration to the repository's default branch. A push runs tests only; it does not start market processing.
4. Open **Actions → Monthly risk run → Run workflow**, select the branch, enter a month as `YYYY-MM` or leave it blank to use the previous calendar month, then select **Run workflow**.
5. Open the resulting run to follow the test, configuration, and per-market jobs. A failing market job does not cancel the other market jobs.

The `monthly-risk.yml` workflow runs tests on pushes to `main`. The manual dispatch reads the enabled markets from `.github/markets.yml` and starts one isolated matrix job for each enabled market. Each job runs Extract, Self Reporting, GPS, Tech Blocklist, and Product Misuse in sequence; different markets run simultaneously. Voice is excluded. There is no scheduled market run currently. ZA is enabled by default and KE is disabled in this example:

```yaml
markets:
  - country: za
    label: SA
    enabled: true
  - country: ke
    label: KE
    enabled: false
```

Only ZA is enabled by default. KE is listed as disabled until its access and data flow have been validated. The matrix loader rejects invalid and duplicate market identifiers, and fails if no market is enabled.



