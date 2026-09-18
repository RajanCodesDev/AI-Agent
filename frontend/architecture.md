# Frontend Architecture

React + Vite + TypeScript SPA talking to the FastAPI backend. No server-side
rendering; all agent output arrives over SSE.

## Structure

```
src/
  main.tsx
  api/            # typed fetch client + auth token handling + SSE client
  auth/           # auth context (token, user, login/logout)
  pages/
    Login.tsx
    Dashboard.tsx      # server table + "Add server" button
    Chat.tsx
  components/
    ServerForm.tsx     # shared form: name, ip, port, username, password-or-key
    ServerRow.tsx
    ChatFeed.tsx
    ToolCallBanner.tsx # shows active ssh tool calls as they stream
  store/          # Zustand: auth, servers, chat state
  types.ts        # shared TS types (Server, ChatMessage, SSE events)
```

## Routing (React Router)

- `/` -> Dashboard (protected), `/chat` -> Chat (protected), `/login` (public).
- Route guard redirects unauthenticated users to `/login`.

## Auth

- On login, store JWT (in memory + localStorage) in the auth store; attach to
  every API request via the fetch client. 401s force logout.

## Security: credentials never touch the chat

- `ServerForm` is the **only** place a password/key is typed. It POSTs directly
  to `POST /api/servers`. Its values are never added to any chat message or
  agent prompt.
- Credential values never appear in a UI event, chat message, agent prompt, or
  tool result.
- The same form is used from two places: the Dashboard "Add server" button and a
  modal opened by the agent's `open_add_server_form` event while chatting.

  ```
  agent -> open_add_server_form -> SSE/UI event -> React opens ServerForm
    -> ServerForm POSTs credentials directly to /api/servers
  ```

## SSE chat flow

1. `Chat.tsx` sends `POST /api/chat/stream` with `fetch()`.
2. FastAPI replies with a `StreamingResponse` of `text/event-stream`.
3. The frontend consumes the response body as a `ReadableStream` and parses SSE
   events (a small SSE parser over `getReader()`).
4. Events: `text` (assistant tokens), `tool` (tool name/status), `done`, `error`.
5. `ChatFeed` renders assistant text; `ToolCallBanner` animates active tool
   calls. Both append into the persisted message list returned by the backend.

## State (Zustand)

- `authStore`: token, user, login/logout actions.
- `serversStore`: list/fetch/add/remove, bootstrap status polling.
- `chatStore`: current session id, messages, streaming flag.

## Bootstrap UX

- Dashboard row shows un-bootstrapped servers with a "Bootstrap" action; state
  comes from a polled `tasks`/server status endpoint while running.