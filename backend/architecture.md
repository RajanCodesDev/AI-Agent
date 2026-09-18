# Backend Architecture

FastAPI monolith. Layered so the agent tools can never bypass ownership checks.

## Layers

```
HTTP
  -> Services
       -> Agent / LangGraph
            -> Agent tools
                 -> Owner-scoped server service
                      -> Credential vault
                      -> SSH executor
                           -> Linux server


LLM-visible:
- friendly server names
- commands
- safe command results
- tool status

Backend-only:
- owner_id
- host/IP
- username
- passwords
- private keys
- Fernet key
- decrypted credentials
```


## Auth & multi-tenancy

- `users` table. Login via JWT (short-lived access token).
- Every `server` row carries `owner_id`. All reads/writes go through
  `Server.query().filter_by(owner_id=current_user.id)`.
- The authenticated user id is injected into every LangGraph run (via
  `configurable`). Tools resolve server names -> `server` records *through the
  session-scoped repository only*. Resolution happens at query time, never as a
  global lookup followed by a later ownership check:

  ```
  repository.get_server(owner_id=current_user.id, name=server_name)
  ```

  A server belonging to another user resolves exactly like an unknown server:
  the tool returns "unknown server" and falls back to the owner-scoped list. It
  never falls back to global scope, and it never leaks the existence of another
  user's server.

## Credential rule (never to the LLM)

- "Add server" form posts straight to `POST /api/servers`. The request body
  (name, host, port, username, password-or-key) is handled by the service layer.
- Credential plaintext never enters a chat message, tool input, tool result,
  LangGraph state, trace span, or log. The system prompt tells the model
  credentials do not exist in its world.
- Passwords are encrypted with Fernet before writing to SQLite. Envelope key
  from `AGENT_ENCRYPTION_KEY` (auto-generated to `data/.env`, chmod 600).
- The credential boundary covers **tool input and tool output alike**:
  - never enter prompts
  - never enter tool arguments
  - never enter tool results
  - never enter LangGraph state
  - never enter chat messages / SSE payloads
  - never enter logs
  - never enter Langfuse traces
  - private SSH keys never enter model-visible state
- Command output returned by `ssh_run` is treated as potentially sensitive.
  The executor/tool layer must not intentionally expose credentials or private
  keys through its output.

## Agent SSH key lifecycle

The backend owns a single Ed25519 agent keypair, generated once and reused
across all servers:

1. Backend generates the agent Ed25519 keypair if it does not exist.
2. Bootstrap authenticates with the registered server password.
3. Bootstrap installs the already-existing public key into the server.
4. Bootstrap verifies key-based login and passwordless sudo.
5. Server is marked `bootstrapped`.
6. All future agent SSH uses the backend-owned private key only.

The LLM never receives or accesses the private key — it only ever invokes
operations that use the key under the hood. This is an MVP key lifecycle; no
PKI, per-server keypairs, or CA infrastructure in scope for v4.

## Data model (core tables)

| Table    | Purpose                                     |
| -------- | ------------------------------------------- |
| users    | auth identity                               |
| sessions | chat session (owner_id, title)              |
| messages | one message per row (session_id, role, content) |
| servers  | owner_id, name, host, port, username, password_encrypted, bootstrapped, fingerprint |
| tasks    | long-running operation results (bootstrap etc.) |

## Bootstrap (one-time, with password)

Uses the backend-generated keypair from the lifecycle above:

1. Connect with registered user + password; establish and record the host key
   into `known_hosts` (this is the initial trust point).
2. Create dedicated system user `devopsagent` (idempotent).
3. Install the agent's Ed25519 pubkey into its `authorized_keys` (0600).
4. Write `/etc/sudoers.d/devopsagent` = `devopsagent ALL=(ALL) NOPASSWD: ALL`
   (0440, validated with `visudo -cf`).
5. Verify key login + `sudo -n true`. Mark `bootstrapped`.

## SSH executor

- Key-only paramiko client; never accepts a password.
- `known_hosts` strictly enforced. The host key is trusted once, at bootstrap;
  every subsequent connection compares against `known_hosts` and **rejects
  unknown or changed host keys**. Host-key verification is never silently
  disabled or downgraded.
- `run(client, command, sudo=True)` wraps privileged commands.

## Agent (LangGraph)

- Primitives only: `list_servers()` (scoped to current user) and
  `ssh_run(server, command, use_sudo)`.
- The temp-key A->B transfer workflow is orchestrated by the model later via a
  skill, using repeated `ssh_run` calls — not a hardcoded tool.
- Graph runs under a per-user thread id; state kept in an in-memory/sqlite
  checkpointer.

## Chat & streaming

- `SSE /api/chat/stream` (sse-starlette). User messages are persisted, agent
  turns (text + tool events) are streamed and stored.
- Langfuse tracing wired with `user_id` as the session tag (self-hosted).
- `open_add_server_form` is a UI event, not an infrastructure primitive.

  ```
  agent -> open_add_server_form -> SSE/UI event -> React opens ServerForm
    -> ServerForm POSTs credentials directly to /api/servers
  ```

  Credentials never appear in the UI event, chat message, agent prompt, or
  tool result.

## Tasks

- Bootstrap runs as a tracked `task` row so the UI can poll status; results
  stored without secrets.