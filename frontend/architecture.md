# Frontend Architecture

React + Vite + TypeScript SPA talking to the FastAPI backend. No server-side
rendering; all agent output arrives over SSE.

The authenticated experience is a persistent **three-pane workspace** (see
`frontend/ui.md` for the visual/interaction contract). There is no Dashboard
page: the chat is the primary workspace and the server manager lives in the
right pane of the same screen.

## Structure

```
src/
  main.tsx
  api/            # typed fetch client + auth token handling + SSE client
  auth/           # auth context (token, user, login/logout)
  store/          # Zustand: auth, sessions, servers, chat state
  types.ts        # shared TS types (Server, ChatSession, ChatMessage, SSE events)
  components/
    AppShell.tsx          # three-pane shell: ChatSidebar | ChatWorkspace | ServerPanel
    Sidebar/
      SessionList.tsx
      NewChatButton.tsx
    Chat/
      ChatHeader.tsx
      ChatFeed.tsx
      ToolCallBanner.tsx   # compact live status for active ssh tool calls
      ChatComposer.tsx
      ServerFormModal.tsx  # shared modal (button + open_add_server_form)
    servers/
      AddServerButton.tsx
      ServerCategory.tsx
      ServerItem.tsx
```

## App shell layout

```
AppShell
├── ChatSidebar     (left, collapsible)     — session navigation
├── ChatWorkspace   (center, primary)       — active chat + composer
└── ServerPanel     (right, collapsible)    — server/connection manager
```

Each pane holds independent state (expanded/collapsed, active session, scroll,
streaming). Switching sessions or server UI state never reloads the app.

## Routing (React Router)

- `/login` (public)
- `/` (protected): the authenticated workspace — session list, active chat,
  and server manager in one route. No separate dashboard route.
- Route guard redirects unauthenticated users to `/login`.

## Auth

- On login, store JWT (in memory + localStorage) in the auth store; attach to
  every API request via the fetch client. 401s force logout.

## Security: credentials never touch the chat

- `ServerFormModal` is the **only** place a password/key is typed. It POSTs
  directly to `POST /api/servers`. Its values are never added to any chat
  message or agent prompt.
- Credential values never appear in a UI event, chat message, agent prompt, or
  tool result.
- The same modal opens from two places: the `AddServerButton` in the right pane
  and an `open_add_server_form` UI event from the agent while chatting.

  ```
  agent -> open_add_server_form -> SSE/UI event -> React opens ServerFormModal
    -> ServerFormModal POSTs credentials directly to /api/servers
  ```

## Server manager (right pane)

- Lists the user's servers as expandable groups (presentation-only
  categorization over server data; no backend category feature unless added).
- `ServerItem` shows friendly name, host/IP, and bootstrap/connection state —
  never credentials.
- Bootstrap action + progress shown per un-bootstrapped server; state from a
  polled `tasks`/server status endpoint.
- Selecting an item does not navigate away from the active chat.

## SSE chat flow

1. `ChatWorkspace` sends `POST /api/chat/stream` with `fetch()`.
2. FastAPI replies with a `StreamingResponse` of `text/event-stream`.
3. The frontend consumes the response body as a `ReadableStream` and parses SSE
   events (a small SSE parser over `getReader()`).
4. Events: `text` (assistant tokens), `tool` (tool name/status), `done`,
   `error`, and the `open_add_server_form` UI trigger.
5. `ChatFeed` renders assistant text (streaming updates in place);
   `ToolCallBanner` shows a compact live status for active tool calls.

## State (Zustand)

- `authStore`: token, user, login/logout actions.
- `sessionsStore`: session list, active session id, new/select session.
- `serversStore`: server list/groups, add/remove, bootstrap status polling.
- `chatStore`: active session messages, streaming flag, tool activity.