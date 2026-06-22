---
name: maxcompute-dataworks-flow
description: Use when working in the homay maxcompute_agent project on MaxCompute connections, PyODPS queries, SQL file execution, DataWorks node SQL sync, table logic refreshes, or data_warehouse SQL changes that must start from online DataWorks logic.
---

# MaxCompute DataWorks Flow

## Overview

Use the repository's established MaxCompute and DataWorks entry points instead of rediscovering connection code. Keep online DataWorks SQL as source of truth before editing local `data_warehouse/` logic.

## First Checks

1. Confirm the repo root, normally `/Users/tsunami/Documents/homay/maxcompute_agent`.
2. Read the local `AGENTS.md` rules before editing SQL under `data_warehouse/`.
3. Do not expose credentials. Treat `.env` and shell environment values as secret.
4. If explaining implementation internals or choosing the right command, read `references/project-map.md`.

## MaxCompute Query Flow

Use the existing PyODPS wrapper:

```python
from maxcompute_client import MaxComputeClient

client = MaxComputeClient(env="prod")
instance = client.execute_sql(sql)
with instance.open_reader() as reader:
    rows = [record.values for record in reader]
```

Prefer the bundled command runner for SQL files:

```bash
python3 maxcompute_tools/execute_sql_script_mode.py \
  -f path/to/query.sql \
  --env dev \
  --individual-only
```

Use `--env prod` only when the user explicitly wants production execution. Add `-v key=value` for `${var}`, `#{var}`, or `{var}` substitutions.

For read queries against partitioned ODPS tables, require explicit single-partition predicates on source tables, normally `t.pt = '${bizdate}'`. Do not introduce partition ranges or `MAX_PT()` anchors unless the user explicitly approves that behavior.

## DataWorks SQL Sync Flow

Before changing any file under `data_warehouse/`, sync the matching online DataWorks node:

```bash
python sync_table_logic_from_dataworks.py --table <table_name>
```

If the shell has no `python`, use the equivalent `python3` command and mention the substitution. For preview only:

```bash
python3 sync_table_logic_from_dataworks.py --table <table_name> --dry-run
```

The sync command scans `data_warehouse/<layer>/<table>/<table>.sql`, calls DataWorks `ListNodes`, matches node names to table names, calls `GetNodeCode`, then writes the returned SQL into the local table SQL file.

Use broader sync tools only when requested:

```bash
python3 sync_all_tables_from_maxcompute.py --env prod
python3 sync_dataworks_nodes.py --full
```

## Validation And Reporting

After query or SQL changes, run the narrowest useful verification command. For warehouse development, prefer:

```bash
python3 maxcompute_tools/execute_sql_script_mode.py \
  -f path/to/main.sql \
  --env dev \
  --validate <target_table> \
  --partition <bizdate> \
  --pk-fields field1,field2
```

Report the command run, environment, changed files, and any unresolved access/configuration issue. If execution is not possible because credentials or `.env` are missing, say that directly and provide the exact command the user can run after configuring secrets.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Editing `data_warehouse/` SQL from stale local files | Run `sync_table_logic_from_dataworks.py --table <table>` first |
| Running production SQL by default | Default to `dev`; use `prod` only on explicit request |
| Reading partitioned tables without `pt` equality | Add `source_alias.pt = '${bizdate}'` on each partitioned source |
| Printing secrets while debugging config | Show variable names and presence/absence only |
| Reimplementing DataWorks API calls | Use the repository sync scripts unless they are broken |
