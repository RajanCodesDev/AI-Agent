# Frontend Plan

Milestones in order. Each is verifiable in the browser. Do not start `Fx`
before `Fx-1` is done. Build the three-pane workspace incrementally as defined
in `frontend/ui.md`.

## F0 — Scaffold

- Vite + React + TS app (`npm create vite`), React Router, Zustand, API client.
- Proxy `/api` to the backend in dev.
- Application shell: three-pane layout (`AppShell`) with ChatSidebar,
  ChatWorkspace, ServerPanel placeholders.

## F1 — Auth & workspace routing

- Login page wired to `/api/auth/login`; JWT stored; route guard on `/`.
- Verify: unauthenticated hit to `/` redirects to `/login`.

## F2 — Chat/session workspace

- Session list in the left pane (active session visually distinct, new chat,
  truncation, independent scroll).
- Active chat in the center: message rendering with distinct user/assistant
  treatment, empty-state greeting ("Hi, how can I help you?"), anchored
  composer.
- Verify: messages scroll, composer stays anchored.

## F3 — Server manager

- Right pane: header with "Add server", expandable/collapsible server groups,
  ServerItem showing friendly name, host/IP, bootstrap state.
- No credentials displayed. Selecting an item does not leave the chat.
- Bootstrap action + progress for un-bootstrapped servers.

## F4 — ServerForm

- Shared `ServerFormModal` (name, ip, port, username, password-or-key).
- Opens from `AddServerButton` **and** the `open_add_server_form` UI event.
- Posts only to `POST /api/servers`. Confirm: submitted credentials never
  appear in the chat feed, agent prompts, SSE events, tool results, logs, or
  traces.

## F5 — Chat SSE

- Chat request sent with `fetch()` to `/api/chat/stream`; consume the
  `text/event-stream` response as a `ReadableStream`.
- Consume `text` / `tool` / `done` / `error` events + `open_add_server_form`
  trigger; streamed assistant text updates in place; tool-call banner shows
  live status.
- "Add server" modal trigger from chat works end to end.

## F6 — Polish

- Collapse/expand behavior for both side panes with reopen affordances.
- Responsive behavior on narrower screens (drawers per priority: chat →
  sessions → servers).
- Loading / error / empty states; logout; bootstrap progress feedback; final
  visual refinement against `frontend/ui.md`.