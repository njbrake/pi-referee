# Changelog

## 0.1.0 (unreleased)

- Review each tool call with a separate reviewer model that sees only your latest message, the agent's reasoning right before the call, and the call.
- Plain-language policies (`safe`, `correct`, and optional ones) toggled per session with `/guard policies`.
- Always-ask patterns for catastrophic commands and sensitive paths.
- Reviewer `block` verdicts ask you by default (`onBlockVerdict`).
- `/guard config` editor for policies, patterns, reviewer model, and settings.
- Status line with live review state, counts, and clickable segments in fullscreen mode.
- Decision log at `~/.pi/agent/pi-referee/reviews.jsonl` for tuning.
