# Configuration, CLI and CI

## Project layout

```
your-project/
  features/               .feature and .saffron files
  saffron.config.json
  .saffron/
    cache/                committed replay caches      → commit
    proposals/            pending AI proposals         → review, then gone
    history.jsonl         one line per run (trends)    → commit recommended
    reports/              latest.html / latest.json    → git-ignore
```

## `saffron.config.json`

```json
{
  "baseURL": "https://stage.your-app.com",
  "features": "features",
  "actionTimeoutMs": 5000,
  "pollIntervalMs": 100,
  "retries": 1,
  "model": "claude-sonnet-5",
  "healModel": "claude-haiku-4-5",
  "maxTurns": 100,
  "storageState": ".auth/state.json",
  "strict": false,
  "assertionPolicy": "strict",
  "verifyProposals": true,
  "reuseSteps": true,
  "snapshotMode": "none",
  "browser": "chromium",
  "workers": 1
}
```

| Key | Meaning |
|---|---|
| `baseURL` | App under test; steps say "the login page", not full URLs |
| `storageState` | Playwright storage-state JSON so replays and the agent start authenticated (`npx playwright open --save-storage=.auth/state.json <url>`) |
| `actionTimeoutMs` | Budget for one action: Playwright's actionability wait and the deadline for a polled assertion (default 5000) |
| `pollIntervalMs` | How often a polled assertion re-checks the page inside that budget (default 100, minimum 10) |
| `retries` | Extra attempts per action with backoff before a step fails and the agent heals (default 1). Not a scenario re-run; reports show a `retried ×N` chip |
| `strict` | Yellow (passed-with-adaptation) exits 1 until reviewed: cached-green-only CI |
| `assertionPolicy` | `strict` (default) or `adaptable-mid`; the final assertion block is always strict |
| `verifyProposals` | Proof-replay every recording zero-AI before filing (default true) |
| `reuseSteps` | Seed new recordings from existing step recordings (default true) |
| `snapshotMode` | `none` (default, ~60% fewer AI calls) or `full` for highly dynamic pages |
| `browser` | `chromium` / `firefox` / `webkit`: replay runs anywhere; recording and healing use Chromium |
| `workers` | Parallel replay workers; agent work stays sequential |
| `healModel` | Cheaper model for heal sessions only |

## CLI

| Command | Purpose |
|---|---|
| `saffron run [paths] [--filter @tag] [--no-agent] [--strict] [--browser b] [--workers n] [--heal-model m] [--headed]` | Run; cached replay, agent on misses/failures |
| `saffron accept [files... \| --all] [--include-unverified] [--with-feature-edit] [--propagate]` | Promote proposals (`--all` skips UNVERIFIED ones unless `--include-unverified`); `--with-feature-edit` rewrites adapted steps in the feature file; `--propagate` applies a heal's locator fix to every cache using that locator |
| `saffron reject [files... \| --all]` | Discard proposals |
| `saffron prune [--yes] [--check] [--json]` | List recordings no scenario owns any more; `--yes` deletes them, `--check` exits 1 for CI |
| `saffron status [--json]` | Project overview: files and scenarios with cache state, tags, pending proposals, last run, history, vocabulary health, effective config |
| `saffron steps [search] [--json] [--snippets]` | The step vocabulary with recorded/divergent/unrecorded badges |
| `saffron author <prose-file>` | Draft a `.saffron` file from plain-paragraph requirements using the project vocabulary |
| `saffron mcp` / `saffron lsp` | MCP tools (`search_steps`, `list_step_sets`, `project_status`) for AI assistants / language server for editors |
| `saffron init [--examples] [--agents list]` | Install this skill into the project's agent directories, register MCP, add an AGENTS.md block, scaffold config; `--examples` adds the Saucedemo example suite |
| `saffron report` | Open the latest HTML report |

Exit codes: 0 green/yellow, 1 red (or yellow with `--strict`), 2 usage /
preflight (e.g. a missing `{env:VAR}`).

## AI access

Recording and healing need Claude credentials: `ANTHROPIC_API_KEY`, or a
Claude Code login on the machine. Replay-only runs (`--no-agent`) need
none: that is the normal CI mode once caches are committed. With no
`model` configured the Agent SDK's default model is used; set `model`
(and `healModel`) to pin one. On a subscription login the report shows
the 5-hour plan window used and the API-equivalent dollars; on a key it
shows what was billed.

## IDE integration

The VS Code extension and the JetBrains plugin add right-click Run on
`.saffron` files, a Saffron panel (feature files, tags, proposals to
accept or reject, vocabulary health, config) and the report as an
in-editor dashboard, all reading `saffron status --json`. Nothing there
is required for agents; the CLI is the same surface.

## Recommended CI

```bash
npx playwright install chromium
npx saffron run --no-agent --strict     # replay committed caches; no AI, no surprises
```

Record and heal on developer machines (or a dedicated job with
credentials), review proposals in PRs like snapshot updates.

## Reading a report

Per scenario: status, duration, AI calls and tokens (including prompt
cache reads/writes: the real bill), adaptation narrative, cache diff and
suggested feature edit for yellows, drift chips when page fingerprints no
longer match. Trends vs. the previous run and 20-run sparklines come from
`.saffron/history.jsonl`; **chronic** scenarios (healing repeatedly) are
flagged: re-record those instead of paying for heals again.
