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
