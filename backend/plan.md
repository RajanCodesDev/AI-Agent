# Backend Plan

Milestones in dependency order. Each milestone is complete and testable on its
own. Do not start `Mx` before `Mx-1` is done.

## M0 — Scaffold

- FastAPI app + uvicorn entry, SQLAlchemy engine (SQLite in `data/devops.db`).
- Config via env (`app/config.py`), data dir, `.env.example`.
- Health endpoint + pytest smoke.

## M1 — Auth & users

- `users` table, signup/login, JWT issuance and dependency.
- `/api/auth/*` endpoints; current-user dependency for all routes.
- Tests: auth round-trip, protected route rejects anonymous.

## M2 — Server CRUD + encrypted vault

- `servers` table with `owner_id`; CRUD `/api/servers` (scoped to owner).
- Fernet encrypt/decrypt (`app/crypto.py`) for passwords/keys; key auto-gen.
- **Credential rule enforced here:** plaintext never in logs/prompts/traces.
- Tests: ownership isolation (user A cannot read/delete B's server).

## M3 — Bootstrap worker

- Generate the backend agent Ed25519 keypair if it does not already exist.
- Bootstrap initially authenticates using the registered password.
- Establish/trust the server host key during bootstrap.
- Create dedicated system user `devopsagent` (idempotent).
- Install the agent public key into `authorized_keys`.
- Configure and validate `/etc/sudoers.d/devopsagent`.
- Verify key login + `sudo -n true`.
- Mark server bootstrapped.
- Password is used only internally by the bootstrap worker.
- The agent never receives the bootstrap password.

## M4 — SSH executor

- `app/ssh.py`: key-only Paramiko executor using the backend-owned Ed25519
  private key.
- Enforce `known_hosts`; reject unknown or changed host keys.
- Reject password authentication paths for normal agent execution.
- Implement `run(..., sudo)`.
- Verify unknown-host connections are refused.
- Private key never enters agent state, tool results, logs, or traces.

## M5 — Agent tools (LangGraph), scoped

- `app/agent/`: LLM factory (anthropic / openai-compatible), graph via
  `create_agent`.
- Tools: `list_servers()` (owner-scoped), `ssh_run(server, command, sudo)`.
- User id threaded into the run via `configurable`; tools reject foreign/absent
  names. System prompt sets the no-credentials rule.
- Tests: tool cannot reach another user's server even when asked by name.

## M6 — Persistent chat + SSE

- `sessions`, `messages` tables; session list/create; restartable threads.
- `SSE /api/chat/stream`: stream agent text + tool events, persist each turn.
- System greeting ("Hi, how can I help you?").
POST /api/chat/stream
→ StreamingResponse(text/event-stream)

Events:
text
tool
done
error

## M7 — Langfuse tracing

- Self-hosted Langfuse via docker-compose; callbacks on the graph run tagged by
  `user_id`. Sanitize tool/trace inputs (no credentials).

## M8 — Hardening

- Key rotation, sudoers re-check on demand, audit log of agent actions, task
  retention/cleanup, rate limits on auth.

## Stretch (after M8, only on request)

- Skill for the temp-key A->B transfer workflow (model-orchestrated, destroyed
  immediately after), documented as a skill not a tool.