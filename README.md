# Risk_Automate_Md
## Overview
This document describes the implemented Extract → analysis flow for Self-Reporting, GPS, Tech Blocklist, and Product Misuse. **Voice is intentionally
excluded** until the table can be built from the **1-Many Dashboard** warehouse raw table/query is built. The idea is to automate the risk flow of how analysis is been done within the risk

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

## What the Extract produces
Run the full monthly warehouse extraction:

```powershell
python -m extract.run_extract --month 2026-09
```

Use `--no-upload` to write local Extract files only:

```powershell
python -m extract.run_extract --month 2026-09 --no-upload
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
python -m extract.run_extract --month 2026-09 --self-reporting-only
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
python -m src.self_reporting.run_self_reporting --month 2026-09
```
Add `--no-upload` to keep the workbook local. Outputs are saved under `data/<COUNTRY>/<YYYY-MM>/self_reporting/` and published under the configured Self Reporting Drive path.



## GPS

GPS does **not** have a separate GPS warehouse query. It uses the monthly `Rafiki_Journey_Raw_Data.csv` and `Tiko_Miles.csv` produced by the standard Extract. The analysis compares facility and offer coordinates using the
Haversine distance. Its default deduction rule is distance **greater than 500 m** with geolocation accuracy **at most 100 m**.

```powershell
python -m src.gps.run_gps_summary --month 2026-09
```
The command reads both source files from that month’s local Extract folder and writes a workbook under `data/<COUNTRY>/<YYYY-MM>/gps/`. To use custom CSVs, provide the raw file positionally and optionally a Tiko Miles file:

```powershell
python -m src.gps.run_gps_summary "C:\path\journey.csv" --tiko-miles "C:\path\tiko_miles.csv" --month 2026-09
```

If a Google Sheet should receive the results, provide its ID with `--sheet-id`. The default run writes a local workbook; it does not create a Drive output folder or upload automatically.

## Tech Blocklist

Tech Blocklist has no separate warehouse SQL. It consumes three files created by the standard Extract:

* `Raw_Data__raw_Auto_blocklist.csv` as `Raw`
* `Raw_Data__Match_Count.csv` as `Pivot Table 5`
* `Raw_Data__Tiko_Miles.csv` as `Tiko_Miles`

The analysis builds suspension and deduction tabs from these sources. The suspension threshold is more than 20 pre-merge profiles; the deduction rule uses a merge count of at least 5.

```powershell
python -m src.tech_blocklist.TechBlocklist --month 2026-09
```

This writes the workbook locally under `data/<COUNTRY>/<YYYY-MM>/tech_blocklist/`. Add `--upload` to publish it to the configured Tech Blocklist Drive path:

```powershell
python -m src.tech_blocklist.TechBlocklist --month 2026-09 --upload
```
