<div align="center">

# pi-referee

**A second model referees your [pi](https://pi.dev) coding agent's tool calls.**<br>
Safe calls run. Risky, mistaken, or off-task calls come to you.

[![CI](https://github.com/njbrake/pi-referee/actions/workflows/ci.yml/badge.svg)](https://github.com/njbrake/pi-referee/actions/workflows/ci.yml)
[![CodeQL](https://github.com/njbrake/pi-referee/actions/workflows/codeql.yml/badge.svg)](https://github.com/njbrake/pi-referee/actions/workflows/codeql.yml)
[![npm](https://img.shields.io/npm/v/pi-referee?color=cb3837&logo=npm)](https://www.npmjs.com/package/pi-referee)
[![pi package](https://img.shields.io/badge/pi-package-6c5ce7)](https://pi.dev/packages)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

<img src="https://raw.githubusercontent.com/njbrake/pi-referee/main/docs/demo.gif" alt="pi-referee in pi: a reviewed command runs, a force push asks for approval, and the status line is clicked" width="820">

</div>

pi-referee is built for running pi on a model you only partly trust (for example, a local model) while a fast, cheap model keeps an eye on it.

## Features

- **Per-session policies.** Plain-language rules like `safe` and `correct`, switched on or off for each session.
- **Escalation, not silent blocking.** Doubtful calls ask you, with the reviewer's reason.
- **Small reviewer context.** The reviewer sees your latest message, the agent's reasoning before the call, and the call. Nothing else.
- **Always-ask backstop.** Force pushes, `sudo`, piping downloads into a shell, and similar calls never go to the reviewer.
- **Live, clickable status line and dialogs.** Watch reviews happen, and click to change policies, answer approvals, or read recent decisions.
- **Tunable inside pi.** `/guard config` edits policies, patterns, the reviewer model, and settings. Every decision is logged.

## How it works

For each tool call, pi-referee picks one of three paths:

1. **Read-only tools run freely.** `read`, `grep`, `find`, and `ls` skip review by default.
2. **Always-ask patterns come straight to you.** Deleting `/` or your home directory, force pushes, piping a download into a shell, `sudo`, disk formatting, shutdown, and edits to SSH keys or shell startup files never go to the reviewer.
3. **Everything else goes to the reviewer model.** It answers `allow`, `ask`, or `block`:
   - **allow**: the call runs.
   - **ask**: you get a prompt with the reviewer's reason: Allow, Block, or Block and tell the agent why.
   - **block**: by default this also asks you. Set `onBlockVerdict` to `"block"` to refuse the call and send the reason back to the agent.

If the reviewer fails or times out, the call asks you by default. Without an interactive UI (print or JSON mode), anything that would ask is blocked.

### What the reviewer sees

Only three things, so reviews stay fast and cheap:

- your latest message, as the reference for what you want,
- the agent's thinking and message immediately before the call,
- the call itself.

It does not see the rest of the conversation or any tool results.

## Install

```bash
pi install npm:pi-referee
# or from GitHub
pi install git:github.com/njbrake/pi-referee
# or from a local checkout
pi install ./pi-referee
```

Restart pi, then choose a reviewer model:

```
/guard config  →  Reviewer model
```

Pick a fast model from your model list. Until you choose one, calls that need review are handled as reviewer failures (they ask you by default).

## Usage

| Command | What it does |
|---|---|
| `/guard` | Show whether the guard is on, which policies are on, and this session's counts |
| `/guard policies` | Toggle policies for this session |
| `/guard toggle <policy>` | Toggle one policy for this session |
| `/guard recent` | List this session's decisions, with reasons |
| `/guard on` / `/guard off` | Turn the guard on or off for this session |
| `/guard config` | Edit the config inside pi: policies, always-ask patterns, reviewer model, and settings |
| `/guard reload` | Reload the config file |

Per-session choices are saved in the session, so they come back when you resume it.

### Status line

In pi's TUI, a status line sits below the editor:

```
◆ guard  [safe] [correct]  ✓12 ?1 ✗0
```

It shows `◆ reviewing <call>…` while the reviewer works and a short result (`✓ ls · 1.2s`) after each decision. In fullscreen mode (`"tuiMode": "fullscreen"`), the line is clickable:

- **◆ guard** opens a menu: turn the guard on or off, policies, recent decisions, config.
- **A policy tag** opens this session's policy checklist.
- **The counts** open recent decisions.

The dialogs these open (menus, the policy checklist, approval prompts) are clickable too, and keep working with the keyboard.

In other modes the same summary appears as plain text in pi's footer, and dialogs use pi's standard select prompt.

## Policies

Policies are plain-language rules for the reviewer. Built in:

| Policy | Default | Enforces |
|---|---|---|
| `safe` | on | Nothing destructive, irreversible, or disruptive beyond what you asked for |
| `correct` | on | The call is a sensible, correct next step for your request |
| `stay-in-project` | off | Only read or modify files inside the working directory |
| `no-network` | off | No network access |
| `no-installs` | off | No installing or removing packages |
| `no-git-writes` | off | No commits, pushes, rebases, resets, or other history changes |

Add, edit, or remove policies with `/guard config`.

## Configuration

The config lives at `~/.pi/agent/pi-referee/config.json` (under `$PI_CODING_AGENT_DIR` if set) and is created on first run. Edit it with `/guard config`, or by hand followed by `/guard reload`. Invalid files fall back to the defaults with a warning.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Guard on by default for new sessions |
| `reviewerModel` | `""` | `provider/model-id` of the reviewer |
| `reviewerThinking` | `"off"` | `"off"` asks the provider to skip thinking (faster); `"default"` sends nothing. Currently only `anthropic-messages` providers honor `"off"` |
| `timeoutMs` | `30000` | Reviewer timeout |
| `maxTokens` | `400` | Reviewer output limit |
| `onBlockVerdict` | `"ask"` | What a reviewer `block` does: `"ask"` you or `"block"` |
| `onReviewerError` | `"ask"` | What happens when the reviewer fails: `"ask"` or `"block"` |
| `includeUserMessage` | `true` | Send your latest message to the reviewer |
| `maxContextChars` | `4000` | Size limit for each context section |
| `skipTools` | `["read","grep","find","ls"]` | Tools that skip review |
| `log` | `true` | Log every decision |
| `systemPrompt` | built in | Reviewer instructions; active policies are appended |
| `rules` | built in | Policies: `{ id, enabled, text }` |
| `alwaysAsk` | built in | `{ id, tool, pattern }`; bash patterns match the command, other tools match `path` |

## Tuning for your reviewer model

Every model reviews differently, so expect to tune:

- Read `~/.pi/agent/pi-referee/reviews.jsonl`. Each line records the call, active policies, verdict, reason, latency, and the reviewer's raw reply.
- Reword policies when the reviewer is too strict or too lenient. Short, concrete policies work best.
- Use **Always-ask patterns → Test a command or path** in `/guard config` to check patterns.
- Prefer a fast model with thinking off. Reviews add one model call per reviewed tool call.

## Limits

- **This is not a sandbox.** Approved calls run with your user's permissions.
- **The reviewer can be wrong.** It is a model making a judgment. The always-ask patterns are the deterministic backstop.
- **The agent's reasoning is a claim, not proof.** A confused or manipulated agent can argue for a bad call. Your latest message is included as a reference point, and tool results are left out.
- **The decision log contains commands and file paths.** Keep it private.

## Development

```bash
bun test                            # unit tests
pi -e ./extensions/pi-referee.ts    # try the extension for one run
```

See [AGENTS.md](AGENTS.md) for repository guidelines, checks, and the release process.

## License

[MIT](LICENSE)
