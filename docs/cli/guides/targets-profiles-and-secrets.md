# Targets, profiles, and secrets

A single Kirimana project builds into more than one place — a local DuckDB for development, a shared platform for production. Kirimana separates *what* you build (contracts) from *where* you build it (targets), and keeps every credential out of your files (vault). This guide covers all three.

## Targets and profiles

A **target** is a named environment — conventionally `dev`, `test`, `prod`. Each target maps to a **profile** in `kiri.yml` that says which adapter and connection to use. Contracts never mention a target; you select one at run time.

List the targets configured for the active project:

```bash
kiri targets
```

Nearly every build command takes `--target` (or `-t`):

```bash
kiri plan  --target dev
kiri apply --target prod
kiri apply -t test
```

When you omit `--target`, the resolution order is: `--target` flag → the `KIRIMANA_TARGET` env var → the project's `default_target`.

### The active project

Targets belong to a project, so first tell Kirimana which project is active on this host:

```bash
kiri use .            # make the current directory the active project
kiri use              # show the currently-resolved project + how it was resolved
kiri use --clear      # forget the persisted default
```

`kiri use <dir>` validates that `kiri.yml` is present, persists the choice for this host, and prints the `export KIRIMANA_PROJECT=…` line you can put in your shell profile.

## Secrets: the `${vault:...}` reference syntax

Secrets are **never** written into YAML in plaintext. Wherever a config needs a credential, you write a reference and store the value in the vault. CI fails on any detected plaintext secret.

The reference form is:

```
${vault:<id>:<key>}
```

- `<id>` — the secret id, the middle segment; often a path-like grouping such as `db/crm` or `airbyte/salesforce`.
- `<key>` — which value inside that secret, such as `password`, `token`, or `config`.

So a database source's password appears in YAML only as:

```yaml
    password: ${vault:db/crm:password}
```

### Storing and listing secrets

Store a value under `<id>:<key>` in the project's env vault:

```bash
# Prompts for the value (hidden input)
kiri vault set db/crm --key password

# Or read it from an env var — useful in CI. The source env var is read and
# then deleted from the process after writing.
kiri vault set db/crm --key password --from-env DB_CRM_PASSWORD
```

`kiri vault set` writes to `<project>/.env` using the EnvVault naming convention (`<PREFIX>_<NORMALIZED_ID>_<NORMALIZED_KEY>`) — the same lookup path that `${vault:db/crm:password}` resolves at load time. The default prefix is `KIRIMANA`; override it with `--prefix` to match `vault.prefix` in `kiri.yml`.

Take a safe inventory — key names only, values redacted:

```bash
kiri vault list
```

This is the "what secrets do I have wired?" view. It never prints a secret value.

The `${vault:...}` syntax is one form across every backend, so contracts and source configs stay portable — the same reference resolves whether the value lives in a local `.env`, a cloud key vault, or elsewhere.

## `KIRIMANA_*` environment variables

Kirimana reads a family of `KIRIMANA_*` env vars so you can drive it from a shell or CI without flags. The ones you meet most often:

| Variable | Effect |
|---|---|
| `KIRIMANA_PROJECT` | Path to the active project. Set by `kiri use`, or export it yourself. |
| `KIRIMANA_TARGET` | Default target when `--target` is not passed (still below an explicit flag). |
| `KIRIMANA_YML` | Explicit path to the `kiri.yml` to use. |
| `KIRIMANA_ACTOR` | The actor recorded on audited operations. |

Vault secrets are also surfaced as env vars under the vault prefix (default `KIRIMANA`) — that is precisely how `${vault:...}` references resolve at load time.

A typical shell setup:

```bash
export KIRIMANA_PROJECT=/path/to/sales_dw
export KIRIMANA_TARGET=dev
kiri apply           # resolves project + target from the environment
```

## Related reference

- [secrets](../secrets.md) — `kiri vault set`, `kiri vault list`
- [getting started](../getting-started.md) — `kiri init`, `kiri use`, `kiri targets`
- [platform operations](../platform-operations.md) — target-specific operational commands
- [adding a source](adding-a-source.md) — where source credentials get wired via `${vault:...}`
