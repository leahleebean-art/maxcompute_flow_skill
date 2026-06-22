# Project Map

Use this reference when you need concrete file paths, command templates, or implementation details for the homay `maxcompute_agent` repository.

## Configuration

- `config.py`
  - `Settings` reads `.env` plus environment variables.
  - MaxCompute variables: `ODPS_ACCESS_ID`, `ODPS_SECRET_ACCESS_KEY`, `ODPS_PROJECT`, `ODPS_ENDPOINT`.
  - DataWorks variables: `DATAWORKS_ACCESS_KEY_ID`, `DATAWORKS_ACCESS_KEY_SECRET`, `DATAWORKS_ENDPOINT`, `DATAWORKS_REGION_ID`, `DATAWORKS_PROJECT_ID`, `DATAWORKS_WORKSPACE_ID`, `DATAWORKS_PROJECT_ENV`.
  - `settings = Settings()` creates the global config object.

## MaxCompute Query Path

- `maxcompute_client.py`
  - `MaxComputeClient(env="dev"|"prod")` wraps `odps.ODPS`.
  - `env="prod"` forces project `homay_data_prod`.
  - Non-prod uses `settings.ODPS_PROJECT`.
  - `execute_sql(sql)` calls `self.odps.execute_sql(sql)`.
  - Result reads use `instance.open_reader()`.

- `multi_env_maxcompute_client.py`
  - Similar wrapper with optional explicit `project_name`.

- `smart_query/mc_adapter.py`
  - Smart Query read-only path.
  - Defaults to `SMART_QUERY_ODPS_ENV=prod`.
  - Uses `MaxComputeClient` and returns columns/rows from `open_reader()`.

## SQL Execution CLI

- `maxcompute_tools/execute_sql_script_mode.py`
  - Main SQL file runner.
  - Script mode uses hint `{"odps.sql.submit.mode": "script"}`.
  - Fallback/explicit individual mode splits statements and prints SELECT results.
  - Variables: `-v bizdate=20260621`.
  - Validation options: `--validate`, `--partition`, `--partition-field`, `--pk-fields`, `--required-fields`, `--metric-fields`, `--validate-sql-file`.

Example:

```bash
python3 maxcompute_tools/execute_sql_script_mode.py \
  -f data_analysis/check_data_status.sql \
  --env dev \
  -v bizdate=20260621 \
  --individual-only
```

## DataWorks Node Sync Path

- `sync_table_logic_from_dataworks.py`
  - Targeted table logic sync.
  - Scans local `data_warehouse` table directories.
  - Calls DataWorks OpenAPI through Tea OpenAPI.
  - Uses `ListNodes` to fetch nodes and `GetNodeCode` to fetch SQL.
  - Matches node names by exact match, contains table name, or table contains node name.
  - Writes fetched SQL into the local table SQL path.

Commands:

```bash
python3 sync_table_logic_from_dataworks.py --table ads_order_match_input_di --dry-run
python3 sync_table_logic_from_dataworks.py --table ads_order_match_input_di
python3 sync_table_logic_from_dataworks.py --scan-only
```

- `sync_dataworks_nodes.py`
  - Bulk DataWorks node export into `data_warehouse/dataworks_nodes/<project_id>/`.
  - Maintains `.sync_state.json`.
  - Useful for broad node snapshots, not for the AGENTS.md prerequisite before editing table SQL.

```bash
python3 sync_dataworks_nodes.py --full
```

- `sync_all_tables_from_maxcompute.py`
  - Lists all MaxCompute tables with `client.odps.list_tables()`.
  - Creates missing `data_warehouse/<layer>/<table>/<table>.sql` placeholders.
  - Calls `sync_table_logic_from_dataworks.run_sync()` for all scanned tables.

```bash
python3 sync_all_tables_from_maxcompute.py --env prod --list-only
python3 sync_all_tables_from_maxcompute.py --env prod
```

## Lineage Helpers

- `maxcompute_tools/get_table_lineage.py`
  - `--mode dw` uses DataWorks lineage through `AliyunServiceClient`.
  - `--mode sql` infers lineage from recent MaxCompute instances.
  - `--include-node-code` can fetch matching node code when a DataWorks project id is supplied.

## Required Local Guardrails

- Before editing `data_warehouse/` SQL, run:

```bash
python sync_table_logic_from_dataworks.py --table <table_name>
```

- Use `python3` only when `python` is unavailable.
- For partitioned ODPS source tables, use explicit single `pt` equality on the source table.
- Do not use partition ranges or `MAX_PT()` as default business-window logic unless explicitly approved.
