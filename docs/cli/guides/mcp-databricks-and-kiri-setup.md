# Wiring MCP: Databricks and Kiri as tools for your assistant

Kirimana has **no web UI** — your interface is a conversation with an LLM
assistant (Claude Desktop, Claude Code, Cursor, …). Two Model Context Protocol
(MCP) servers make that conversation powerful:

| MCP server | What it exposes | Who ships it |
|---|---|---|
| **`kiri-mcp`** | Your Kirimana project — catalog, lineage, PII, contracts — as read-first tools and resources | Kirimana (ships with `kiri-cli`) |
| **Databricks MCP** | Your Databricks workspace — `list_catalogs`, `list_schemas`, `execute_sql`, `list_clusters`, … | Third-party ([databricks-mcp]) |

Connect **both** and your assistant can reason about the contract *and* inspect
the live warehouse in one session: *"draft the orders contract, then show me the
row count of the table it built."* This guide wires up each one.

> **Secrets, always.** Tokens go in the MCP client config's `env` block, and
> that file is **gitignored** — never commit a PAT. Per **ADR 0011**,
> Kirimana's own secrets are `${vault:...}` references; the MCP client config is
> the one place a raw token lives, so treat it like an SSH key.

---

## 1. Kiri MCP (`kiri-mcp`) — your project as tools

`kiri-mcp` is a console entry point that ships inside `kiri-cli`. It serves your
project's catalog over stdio, so any MCP-capable assistant can search assets,
read lineage, and list PII columns.

**Verify it's installed:**

```bash
pip install kiri-cli      # kiri-mcp comes with it
which kiri-mcp            # → …/bin/kiri-mcp
```

**Claude Desktop** — add to
`~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "kiri": {
      "command": "kiri-mcp",
      "env": {
        "KIRIMANA_PROJECT_DIR": "/absolute/path/to/my_warehouse"
      }
    }
  }
}
```

`KIRIMANA_PROJECT_DIR` points at the project root (the directory holding
`kiri.yml`); `kiri-mcp` reads the catalog Kirimana maintains under
`.kiri/`. To point at a specific catalog database instead, set
`KIRIMANA_CATALOG_DB=/absolute/path/to/.kiri/catalog.sqlite`.

**Claude Code / Cursor** — the same server goes in the repo's `.mcp.json`
(gitignored):

```jsonc
// <project>/.mcp.json
{
  "mcpServers": {
    "kiri": { "command": "kiri-mcp", "env": { "KIRIMANA_PROJECT_DIR": "." } }
  }
}
```

**What the assistant gets** (read-first in the beta):

- Resources — `catalog://assets`, `catalog://asset/{urn}`, `catalog://pii/scan`,
  `catalog://lineage/{urn}`, `catalog://glossary`.
- Tools — `search_assets(query, limit)`, `list_assets_by_domain(domain)`,
  `list_pii_columns()`.

The write path — `kiri plan`, `kiri apply`, `kiri contract new` — stays on the
CLI. The pattern is: the assistant **reads and drafts** through MCP, then hands
you the exact `kiri` commands to run. Every AI call the CLI makes is still
audited through the gateway (see [AI gateway setup](ai-gateway-setup.md)).

---

## 2. Databricks MCP — your workspace as tools

The Databricks MCP server is a **separate, third-party process** ([databricks-mcp])
that you clone and build locally; the assistant launches it via a `.mcp.json`
entry. It is *not* part of Kirimana — full setup and gotchas live in
**`docs/integrations/databricks-mcp-setup.md`**.
The essentials:

```jsonc
// <project>/.mcp.json  (gitignored — holds a PAT)
{
  "mcpServers": {
    "databricks": {
      "command": "/absolute/path/to/databricks-mcp/.venv/bin/databricks-mcp-server",
      "args": [],
      "env": {
        "DATABRICKS_HOST": "https://adb-XXXXXXXXXXXX.N.azuredatabricks.net",
        "DATABRICKS_TOKEN": "dapiXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
        "DATABRICKS_WAREHOUSE_ID": "0123456789abcdef"
      }
    }
  }
}
```

Three things that save you an afternoon:

1. **`DATABRICKS_WAREHOUSE_ID` is not optional in practice** — without it,
   every `execute_sql` fails with an opaque error. Find it under **SQL
   Warehouses → your warehouse** (the last segment of `/sql/1.0/warehouses/<id>`).
2. **Restart the client fully** after editing `.mcp.json` — it's read once at
   session start; `/mcp reconnect` is not enough.
3. **Scope the PAT** to a service principal with only the catalogs/schemas the
   agent should touch. Give it a scratch catalog, not production, while you
   explore.

---

## 3. Both together — the intended workflow

With `kiri` and `databricks` both connected, a single session can:

```text
You:  Using the kiri catalog, what PII is in the crm domain?
Kiri: (reads catalog://pii/scan) → lists the columns + categories.

You:  Draft a contract for the orders table and tell me the plan.
Kiri: drafts contracts/…/orders.yml, hands you `kiri plan --target prod`.

You:  (run it, apply it) …then via the databricks tools, show the row count
      of the gold table it built.
Kiri: (execute_sql) SELECT count(*) … → confirms the live result.
```

Kiri drafts and reasons; you run the `kiri` commands (or the reference skills in
**`skills/`** that wrap them); the Databricks MCP lets the
assistant verify the live outcome. Nothing bypasses the gateway or the
governance rules.

---

## Related reference

- [AI gateway setup](ai-gateway-setup.md) — configure the provider + keys the CLI uses
- [From first startup to production](first-platform-to-production.md) — the end-to-end runbook
- **`docs/integrations/mcp.md`** — the full `kiri-mcp` reference
- **`docs/integrations/databricks-mcp-setup.md`** — the full Databricks MCP setup + gotchas
- [Databricks & adapters](databricks-and-adapters.md) — targets, profiles, and the Delta adapter

[databricks-mcp]: https://github.com/databrickslabs/mcp
