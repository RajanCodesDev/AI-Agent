# Frontend Plan

Milestones in order. Each is verifiable in the browser. Do not start `Fx`
before `Fx-1` is done.

## F0 — Scaffold

- Vite + React + TS app (`npm create vite`), React Router, Zustand, API client.
- Proxy `/api` to the backend in dev; simple layout shell.

## F1 — Auth & routing

- Login page wired to `/api/auth/login`; JWT stored; route guards on `/` and
  `/chat`.
- Verify: unauthenticated hit to `/` redirects to `/login`.

## F2 — Dashboard + server table

- Fetch and list the user's servers (name, host, user, bootstrap status).
- Delete server; refresh on mutation. Bootstrap status shown.

## F3 — ServerForm (shared, credential-safe)

- Shared `ServerForm` component (name, ip, port, username, password-or-key).
- Opens on Dashboard button **and** as a modal when the agent calls
  `open_add_server_form`.
- Posts only to `POST /api/servers`. Confirm: submitted credentials never
  appear in the chat feed or agent prompts.

## F4 — Chat UI + SSE

- Chat page: session list, persisted history, message input, SSE.
- Send the chat request with `fetch()` to `/api/chat/stream` and consume the
  `text/event-stream` response as a `ReadableStream`.
- Consume SSE `text` / `tool` / `done` / `error` events; render assistant text
  and a tool-call banner.
- "Add server" modal trigger from chat works end to end.

## F5 — Polish

- Loading / error / empty states; logout; responsive layout; bootstrap progress
  feedback. Final pass with a real server against M0–M5 backend.