# Repository Guidelines

> `CLAUDE.md` links to this file. Edit `AGENTS.md` only.

pi-referee is an extension for the [pi coding agent](https://pi.dev). A reviewer
model checks each tool call against policies that are toggled per session, and
calls that need a human come to the user for approval. pi loads the TypeScript
source directly; there is no build step.

## Where to look

- `extensions/pi-referee.ts`: the whole extension. Exported pure helpers sit at
  the top (config, context extraction, prompt building, verdict parsing). The
  default export wires pi events (`session_start`, `tool_call`,
  `session_shutdown`), the status line, dialogs, and the `/guard` command.
- `tests/pi-referee.test.ts`: Bun unit tests for the exported helpers.
- `docs/demo.gif`: the README demo.
- `README.md`: user documentation and the configuration reference.
- pi's extension API docs ship inside the installed pi package under
  `docs/` (`extensions.md`, `tui.md`, `rpc.md`, `packages.md`). Read them before
  using a pi API you have not used in this file.

## Commands

```sh
bun test                                                     # unit tests
bun build ./extensions/pi-referee.ts --target=node \
  --external '@earendil-works/*' --outdir "$(mktemp -d)"     # syntax and import check
npm pack --dry-run                                           # published files
pi -e ./extensions/pi-referee.ts                             # try it for one run
```

When trying the extension, remove any installed copy first so calls are not
reviewed twice. Point `PI_CODING_AGENT_DIR` at a temporary directory to keep a
test run away from your real config and decision log. `pi --mode rpc` can drive
dialogs from a script (see pi's `rpc.md`).

## Code rules

- No runtime dependencies. pi's own packages are peer dependencies: import
  their types statically and load runtime modules (such as
  `@earendil-works/pi-tui`) with a dynamic import, so the tests run outside pi.
- Keep the extension in one file until it clearly outgrows it.
- Put logic that does not need a pi context in exported functions and test it.
- Fail safe. A reviewer error, timeout, or unparseable reply must never allow a
  call silently. Without an interactive UI, anything that would ask is blocked.
- The reviewer sees only the user's latest message, the agent's reasoning
  immediately before the call, and the call. Do not widen that without a clear
  reason, and never add tool results.
- A config change touches `GuardConfig`, `DEFAULT_CONFIG`, `validateConfig`, the
  `/guard config` menu, the README table, and the tests in the same change.
  Existing config files must keep loading; missing keys fall back to defaults.
- User-facing text says "policies". Config keys and the reviewer prompt keep
  "rules" so existing configs stay valid.
- The status line must keep its plain `setStatus` fallback for RPC and print
  modes.
- Show list dialogs with `choose()`, not `ctx.ui.select` directly. It renders a
  clickable `SelectList` in the TUI and falls back to `ctx.ui.select` elsewhere.
- Keep comments short and precise. Do not use em dashes or `--` as prose
  separators in docs and comments.
- Never add personal data, hostnames, API keys, or model names from your own
  setup to code, tests, docs, or commits.

## Tests

Tests must be deterministic and isolated from user state: set
`PI_CODING_AGENT_DIR` to a temporary directory and clean it up. Do not call real
models or the network. Use table-style cases when setup is shared, and do not
test constants or trivial getters.

Behavior that needs pi itself (dialogs, the status line, clicks) is verified
manually or with a scripted RPC session. Describe how you checked it in the PR.

## Pull requests

- Use Conventional Commits for PR titles with a lowercase subject
  (`feat: add no-docker policy`). CI enforces this.
- Follow `.github/pull_request_template.md`. Include a screenshot or recording
  for status line or dialog changes.
- Before review, run `bun test`, the build check, and `npm pack --dry-run`.
- Add user-visible changes to `CHANGELOG.md` under the unreleased section.

## Releases

Bump `version` in `package.json`, move the changelog entries under that
version, and push a `vX.Y.Z` tag. The release workflow runs the tests, checks
that the tag matches `package.json`, and publishes to npm through npm trusted
publishing (OIDC), which also adds provenance. No npm token is stored anywhere;
the trusted publisher on npmjs.com is tied to `release.yml`, so keep that
filename. To retry a failed release, run the Release workflow from the Actions
tab with the release tag selected.

## Stored data

- Config: `<agentDir>/pi-referee/config.json`. Decision log:
  `<agentDir>/pi-referee/reviews.jsonl`. `<agentDir>` is `PI_CODING_AGENT_DIR`
  or `~/.pi/agent`.
- Per-session state is a `pi-referee-state` custom session entry. The legacy
  `pi-guard-state` entry and `pi-guard/config.json` are still read.
- Changes to these paths or formats belong in `loadConfig` and `restore`, not in
  ad hoc fallbacks elsewhere. Never commit config files or decision logs.
