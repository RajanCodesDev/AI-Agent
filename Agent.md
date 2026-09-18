# Agent.md — Coding rules for this repo

These rules bind any coding agent (opencode, clines, etc.) working here.

## Scope discipline

- Work only on the milestone the user points at in `backend/plan.md` or
  `frontend/plan.md`. Do not implement unlisted features, and do not "jump
  ahead" to a later milestone.
- If a change requires stepping outside the current milestone, STOP and tell the
  user. Do not silently expand scope.

## Minimal, clean changes

- Make the smallest focused change that satisfies the milestone.
- Do not refactor unrelated code, rename existing symbols, or reformat files
  you are not touching.
- Match the surrounding conventions: same imports, naming, style, and layout.
- Add no code comments unless the user asks. Name things so they read clearly.
- No dead code, no speculative abstractions, no "nice to have" utilities.

## Dependencies

- Ask before adding any dependency. Off-list additions require approval.
- Prefer stdlib / already-listed packages.

## Security (non-negotiable)

- Never let credentials reach an LLM prompt, tool argument, tool result,
  LangGraph state, chat message / SSE payload, log line, or trace.
- Never query another user's rows. Every server/look-up filters by the
  authenticated user id.
- Never write secrets to files other than the encrypted vault / env.
- Never disable host-key verification silently.
- Tool results must be credential-safe too. Never return passwords,
  private keys, decrypted server records, or credential-bearing command
  output to the model. `ssh_run` command output is potentially sensitive —
  sanitize it at the executor/tool layer.
- SSH private keys are infrastructure-side state and are never part of
  model-visible state. The agent may invoke operations that use them, but may
  never read or expose their contents.

## Verification

- Run the project's tests and linters before finishing; if none exist, say so
  and run at least an import/startup smoke check.
- After finishing, report exactly what changed and any commands to run.

## Docs

- Document plan/architecture changes in the corresponding `plan.md` /
  `architecture.md` when a decision changes. Keep them concise and current.
