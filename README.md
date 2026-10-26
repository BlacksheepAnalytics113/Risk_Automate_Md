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
