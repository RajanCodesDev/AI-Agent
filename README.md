# devops_agent-v4

A **self-hosted, multi-user DevOps dashboard + agent**.

Each user registers their servers once (friendly name, IP, port, username,
password or SSH key). The app bootstraps each server — installing the agent's
Ed25519 key and passwordless sudo — then the agent can SSH into any of *that
user's* servers on demand and orchestrate ops between them, all from a web chat.

## Vision

- **Chat-first workspace in the browser.** Not a CLI, not a traditional
  dashboard. A three-pane layout: chat sessions on the left, active chat in
  the center, server manager on the right. Add servers via the right pane or
  ask the chat to open the form.
- **Multi-user with strict isolation.** Every server belongs to one user. The
  agent can only ever see and touch the servers of the authenticated user.
- **Credentials never reach the LLM.** Server passwords/keys are entered in a
  client-side form that POSTs straight to a REST endpoint. The model only ever
  receives friendly server names. It can never read, echo, or leak a secret.
- **Key-only operations.** After one-time bootstrap, the agent authenticates
  with its own SSH key and never needs a password again.
- **Persistent chats.** Sessions and messages survive reloads.

## Security invariants

1. Passwords are encrypted at rest (Fernet; envelope key in env, `chmod 600`).
2. Passwords are used once, at bootstrap time, server-side — never in a prompt.
3. All post-bootstrap operations authenticate with the agent SSH key only.
4. Server access is always filtered by the authenticated user's id.
5. `sudo` is granted passwordless to the dedicated `devopsagent` system user.

## Tech stack

| Layer   | Choice                                           |
| ------- | ------------------------------------------------ |
| Backend | FastAPI, SQLAlchemy, SQLite, paramiko, Fernet, uv/uvicorn |
| Agent   | LangGraph / LangChain, Langfuse (self-hosted)     |
| Frontend| React (Vite + TypeScript), React Router, Zustand, SSE |
| LLM     | Anthropic or OpenAI-compatible (BYO key)         |

## Docs

- `backend/architecture.md`, `backend/plan.md`
- `frontend/architecture.md`, `frontend/plan.md`
- `Agent.md` — guardrails for any coding agent working in this repo