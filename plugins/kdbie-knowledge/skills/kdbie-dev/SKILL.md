---
name: kdbie-dev
description: Use this skill when the user is working with KX Insights Enterprise (IE) — managing packages or deployments via the `kxi` CLI, or handling IE installation and upgrades via Helm/Kubernetes.
---

# Operating KX Insights Enterprise via kxi

`kxi` is a Python CLI (location varies — often a venv) that drives a _live_ Insights Enterprise
deployment. Every command runs against a specific profile (`~/.insights/config.toml`) and a
Kubernetes namespace, and several actions (`pm deploy`/`teardown`, `install delete`) are
destructive. So before running anything: resolve the active profile and namespace, and never
operate on a namespace the user didn't explicitly name.
The core workflow is **packages**. Two command groups, easy to confuse:

`kxi package` — build/validate/pack a package locally (`init` → `validate` → `packit`),
producing a `.kxi` file. Nothing touches a running instance.
`kxi pm` — the package manager on a running IE instance: `push` (upload), `deploy` (activate),
`list`/`info`, `teardown`. `push` **does not deploy** — they're deliberately separate steps.

(`kxi install` handles the Helm/K8s lifecycle; `kxi obs` retrieves logs.)

## Configuration

KXI_BIN=kxi

The binary is a python executable `kxi`, which may be available on the main path, but allow the user to define a different location as it is a common pattern to install the `kxi` binary in a virtual environment.

### Profile Config File Location

> **IMPORTANT:** The kxi CLI stores all profile configuration in **`~/.insights/config.toml`** (TOML format, kxi ≥ 1.12). There is **no `~/.kxi` directory** — do not reference or attempt to create it.
>
> - Active profile: `current_profile = "default"` field at the top of the file
> - Per-profile settings (hostname, namespace, realm, etc.) under `[profiles.<name>]`
> - Legacy installs (kxi < 1.12) use `~/.insights/cli-config` (INI format) as a fallback
>
> To read the active profile's hostname programmatically:
> ```bash
> python3 -c "import tomllib; c=tomllib.load(open('$HOME/.insights/config.toml','rb')); p=c.get('current_profile','default'); print(c['profiles'][p]['hostname'])"
> ```

The `kxi` binary allows for the user to create and utilise different profiles by appending `--profile <profile>` to the `kxi` command. For example `kxi --profile uat` would issue kxi commands against that uat profile. Profiles are created via the `kxi configure --profile <profile>` command and configuration stored in `~/.insights/config.toml`. To permanently switch the CLI default: `$KXI_BIN workon <profile>`.

### Changing the Profile

To change the active profile for a conversation, the user can say e.g. "use profile staging" or "switch to profile foo". When this happens, update the active profile for all subsequent `kxi` commands in the session. Do NOT permanently rewrite this file — treat it as session state only.

### Namespace Resolution (Required for `kxi install` commands)

`kxi install` subcommands (`get-values`, `upgrade`, `run`, `history`, `rollback`, `delete`) do **not** reliably read the namespace from the profile — they will error with `namespace None` even when the profile has a namespace set. Always resolve and pass `--namespace` explicitly.

**Resolution order:**

1. **Read from profile config** — parse `~/.insights/config.toml` for the active profile's `namespace` field:
   ```bash
   python3 -c "import tomllib; c=tomllib.load(open('$HOME/.insights/config.toml','rb')); print(c['profiles']['<profile>'].get('namespace',''))"
   ```
   If config.toml is absent or the profile predates the toml format, fall back to `~/.insights/cli-config` (INI format):
   ```bash
   awk -F' *= *' '/^\[<profile>\]/{found=1} found && /^namespace/{print $2; exit}' ~/.insights/cli-config
   ```

2. **Ask the user** if namespace is not set; default to `kxi` if they have no preference.

Once resolved, hold the namespace as session state and pass it to every `kxi install` command via `--namespace <ns>`.

---

## Token Expiry — Auto-Refresh and Retry

When any `kxi` command fails with `401 Unauthorized` or `HTTPStatusError: Client error '401 Unauthorized'`, the session token has expired. **Automatically attempt to refresh and retry** — do not just surface the error to the user.

**Steps:**

1. Check grant type from auth status:
   ```bash
   $KXI_BIN [--profile <profile>] auth status
   ```

2. Re-authenticate based on grant type:

   | Grant type | Re-auth command |
   |------------|-----------------|
   | `user` | `$KXI_BIN [--profile <profile>] auth login --force-code` |
   | `serviceaccount` | `$KXI_BIN [--profile <profile>] auth login --serviceaccount` |

   For `user` grant: inform the user that a device code login is needed, run the command, and wait for them to complete the flow before retrying.

3. Retry the original command. Only escalate if re-auth itself fails.

---

## kxi pm — Package Manager (Primary Workflow)

The most used command group. Manages packages on a running IE instance.

> **Push safety rules (always apply):**
> - Never add `--deploy` to `kxi pm push` unless the user explicitly asks to deploy immediately after pushing.
> - Never add `--force` to `kxi pm push` unless the user explicitly asks to force-overwrite an existing package.

```bash
kxi pm list [--filter name=<pattern>]          # List packages
kxi pm info <package> [pipeline|table]         # Package details

kxi pm push <path>                             # Push local package (does NOT deploy)
kxi pm push <path> --deploy                    # Push + deploy (only if user explicitly asks)
kxi pm push <path> --force                     # Overwrite existing (only if user explicitly asks)

kxi pm deploy <package> [version]              # Deploy pushed package
kxi pm deploy <package> --db <name>            # Deploy specific DB
kxi pm deploy <package> --pipeline <name>      # Deploy specific pipeline

kxi pm teardown <package>                      # Teardown deployed package
kxi pm teardown <package> --rm-data            # Teardown and delete data
```

---

## kxi package — Local Package Building

Builds and manages packages locally (not on a running IE instance). Always scaffold with `kxi package init <path>` — do not invent structure by hand.

### Valid IE Package Directory Structure

```
<package-name>/
├── .kxignore              # files to exclude from packaging (auto-created by init)
├── manifest.yaml          # root config — registers all entities
├── init.q                 # default entrypoint (required by convention)
├── router/
│   └── router.yaml        # query routing config (needed when database has a DAP)
├── tables/
│   └── <table>.yaml       # one file per table schema
├── databases/
│   └── <db-name>/
│       └── shards/
│           └── <shard-name>.yaml   # SM/DAP/sequencer/mount config
├── pipelines/
│   └── <pipeline-name>.yaml        # pipeline resource config
└── src/
    └── <pipeline-spec>.{q,py}      # spec file referenced from pipeline YAML (use base: python for .py)
```

#### tables/\<table\>.yaml

```yaml
# yaml-language-server: $schema=https://code.kx.com/insights/enterprise/packaging/schemas/package.json#/$defs/Table
type: partitioned          # or splayed / keyed
prtnCol: <timestamp-col>   # required for partitioned; do NOT include a 'date' col — it is inferred
columns:
- name: <col>
  type: <kdb-type>         # timestamp, symbol, string, int, long, float, boolean, ...
```

> **Partitioned table rule:** never include a `date` column in the schema — IE infers it automatically. Adding it explicitly causes errors or duplication.

#### pipelines/\<pipeline\>.yaml

```yaml
# yaml-language-server: $schema=https://code.kx.com/insights/enterprise/packaging/schemas/package.json#/$defs/Pipeline
type: spec
base: python               # or q
spec: file://src/<spec>.py
destination: <db-name>     # database this pipeline writes to
replicas: 1
maxWorkers: 1              # REQUIRED when using directWrite — multiple workers cause undefined behaviour
minWorkers: 1
controller:
  persistence:
    size: 1Gi
worker:
  persistence:
    size: 10Gi
```

#### Workflow to build and push a new package

```bash
kxi package init <package-name>
kxi package validate <package-name>
kxi package packit  <package-name>      # produces <package-name>-<ver>.kxi
kxi pm push <package-name>              # push only — do NOT add --deploy unless user asks
kxi pm deploy <package-name>            # separate deploy step
```

---

## kxi install — Helm/Kubernetes Installation

> **Always pass `--namespace <ns>`** to every install subcommand — the CLI does not reliably read it from the profile and will error with `namespace None`. Resolve the namespace first (see Namespace Resolution above).

```bash
kxi install run     --filepath <values.yaml> --namespace <ns>                        # Install IE
kxi install upgrade --filepath <values.yaml> --namespace <ns> --version <ver>        # Upgrade IE
kxi install get-values                       --namespace <ns>                        # Show deployed values
kxi install delete                           --namespace <ns>                        # Uninstall IE
kxi install list-versions                                                            # List available IE versions
```

### Install/Upgrade Component Rules

IE installs/upgrades involve three independent components. Understanding when each can be upgraded avoids interactive prompts and aborted upgrades.

> **Namespace safety rule (non-negotiable):** Only ever operate on the namespace explicitly specified by the user. **Never** tear down packages/assemblies, delete IE deployments, or run `kxi install delete` against any namespace other than the one the user asked you to work in. Other namespaces belong to other IE deployments that may be in active use.

#### kxi-operator (cluster-wide)

- Installed/upgraded **once per cluster**, not per namespace. It manages all IE deployments across the cluster.
- **Blocked** if any deployed packages/assemblies exist in **any namespace** on the cluster. The CLI will warn and prompt for confirmation.
- **Decision rule:** Before running an upgrade, check for cross-namespace assemblies:
  ```bash
  kubectl get assemblies --all-namespaces 2>/dev/null || kxi pm list
  ```
  - If assemblies exist in **other** namespaces → **always** use `--skip-operator`. Never touch those namespaces.
  - If unsure → **ask the user** whether the operator should be upgraded.

#### kxi-management-service

Upgraded automatically on every `kxi install upgrade`; skip with `--skip-management`.

#### kxi Insights Enterprise (main chart)

- Packages in the target namespace are automatically backed up, torn down, and redeployed post-upgrade.
- Packages in **other** namespaces are **not** touched by the upgrade.

#### Upgrade flags summary

| Flag | Effect |
|------|--------|
| `--skip-operator` | Do not upgrade the kxi-operator, skip without prompting |
| `--skip-management` | Do not upgrade the management service |
| `--skip-packages` | Do not re-apply packages after upgrade |
| `--skip-tasks` | Do not run management service tasks post-upgrade |
| `--force` | Answer yes to all confirmation prompts (including operator skip prompt) |

---

## kxi obs — Observability

`kxi obs logs` takes **no positional arguments** — the workload is specified via `--workload`.

```bash
kxi obs logs --workload <name>                    # Get logs for a workload
kxi obs logs --workload <name> --level WARN       # WARN and above
kxi obs logs --workload <name> --since-seconds 86400
kxi obs logs --workload <name> --search "string"  # Case-insensitive search
kxi obs logs --workload <name> --watch            # Stream continuously
```

---

## Notes

- `kxi assembly` commands are **deprecated** — use `kxi pm` equivalents instead
- `kxi package remote` is an alias for `kxi pm`
- Use `--yes / -y` on pm commands to skip confirmation prompts in scripts
- Use `--output-format json` on pm commands for machine-readable output (e.g., piping to jq)
- `kxi package lock` respects `/dnc` or `//dnc` on first line of q files (do not compile)
