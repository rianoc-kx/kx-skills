# qlint-knowledge

Claude Code plugin shipping the `/qlint-snippet` skill — lint a single q/kdb+ code snippet using [KX qlint](https://code.kx.com/insights/1.13/qlint/) without writing it to a JSONL first.

Useful for:
- ad-hoc syntax checks on a code block pasted into chat
- tool-use loops where a model self-validates the q it just generated

## Prerequisites

| Requirement | How to satisfy |
|-------------|----------------|
| `q` (kdb+) on `PATH` | Install kdb+; or set the `Q` env var to an absolute path. |
| `QLINT_DIR` env var | Point at the directory that contains KX's `qlint.q_`. Required — there is no default. Without it the wrapper exits 2. |

### Recommended setup

If you don't already have **KX Developer** installed, follow <https://code.kx.com/developer/getting-started/> and install it under `~/developer/`. After installing, verify that `~/developer/ws/qlint.q_` exists, then set:

```bash
export QLINT_DIR="$HOME/developer/ws"
```

Add the line to your `~/.bashrc` so the skill picks it up in every shell.

## Install

From inside Claude Code, with the `kx-skills` marketplace already added:

```
/plugin install qlint-knowledge@kx-skills
```

If you haven't added the marketplace yet:

```
/plugin marketplace add KxSystems/kx-skills
/plugin install qlint-knowledge@kx-skills
```

After install, `/qlint-snippet` appears in the user-invocable skills list and Claude will also auto-trigger it when you ask things like "lint this q code".

## Usage

### As a Claude Code skill

Invoke `/qlint-snippet` from any Claude Code session with the q code to lint, or just ask Claude to lint a snippet — the description-based auto-trigger covers phrases like "qlint this", "check syntax of …", etc. The skill ([`skills/qlint-snippet/SKILL.md`](./skills/qlint-snippet/SKILL.md)) drives Claude through the wrapper invocation and reports findings back.

### Direct shell invocation

The wrapper script lands under your installed plugins cache. To find it:

```bash
RUN_SH=$(find ~/.claude/plugins -path '*qlint-knowledge/skills/qlint-snippet/run.sh' | head -1)

# Preferred for multi-line (quoted heredoc — no shell expansion)
"$RUN_SH" <<'EOF'
getTrades:{[sym;d] select from trades where sym=sym, date=d};
getTrades[`AAPL; .z.d]
EOF

# Single-line argument form
"$RUN_SH" 'select sum size by sym from trades'
```

**Do not** write `'... \n ...'` as a multi-line argument — bash single quotes treat `\n` as two literal characters and q's parser will error on the bare `\`. Use a heredoc, a real newline inside `'...'`, or `$'...\n...'` (ANSI-C quoting).

## Output

The wrapper prints the raw `.qlint.lintItem` table to stdout (one row per finding) with columns: `label`, `errorClass`, `description`, `problemText`, `errorMessage`, `startLine`, `startCol`, `endLine`, `endCol`.

Exit codes:

| Code | Meaning |
|------|---------|
| `0` | No `errorClass=`error` rows. Warnings/notes may still be present. |
| `1` | At least one error-level row. |
| `2` | Environment misconfig (missing `qlint.q_` or `q` binary). |

## Optional: pre-approving the wrapper in your settings

Plugins cannot ship a Claude Code permissions allowlist — that is by design (a plugin shouldn't be able to self-authorise). By default you'll be prompted the first time Claude tries to run the wrapper. If you'd like to skip that prompt, add the following to your own `~/.claude/settings.json` (the path glob covers wherever the plugin actually lives in the cache):

```json
{
  "permissions": {
    "allow": [
      "Skill(qlint-snippet)",
      "Bash(*/qlint-knowledge/skills/qlint-snippet/run.sh)"
    ]
  }
}
```

## Notes

- The wrapper runs KX qlint only and shows every finding raw — no policy filtering. If you want to suppress specific rules, filter the output yourself.
