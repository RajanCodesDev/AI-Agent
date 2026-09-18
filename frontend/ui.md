# Frontend UI — Visual Design & Interaction Contract

The authenticated experience is a persistent **three-pane workspace**, not a
dashboard-plus-chat split. No separate "Dashboard" page as the primary
post-login experience.

Spatial composition follows a developer/operations workspace (server-manager
style reference): dark, minimal, technical, dense, low visual noise.

---

## 1. Primary workspace

```
┌─────────────────────────────────────────────────────────────┐
│ LEFT PANE       CENTER CHAT                 RIGHT SERVERS   │
│ Chat sessions   Active conversation          Add Server     │
│                                                            │
│                 Messages                    Server groups   │
│                 Tool activity               Server cards    │
│                 Composer                                  │
└─────────────────────────────────────────────────────────────┘
```

- **Left:** chat/session navigation
- **Center:** active chat — the primary workspace
- **Right:** server/connection manager

---

## 2. Left pane — chat sessions

- Fixed/constrained width when expanded; collapsible.
- Collapsing grants the center chat more horizontal space; collapsed state
  keeps an obvious reopen affordance.
- Lists all sessions belonging to the authenticated user; active session is
  visually distinct.
- "New chat" action available.
- Titles compact and truncated — never stretch the pane.
- Pane scrolls independently when the session list is long.
- It is navigation, not a second chat area.

---

## 3. Center pane — chat

- Occupies the majority of available width and stays usable when either side
  pane is collapsed.
- Messages vertically scrollable; assistant vs user messages have clearly
  different visual treatment.
- Tool execution renders as a compact live status/banner — no internal
  implementation detail dumps.
- Streaming assistant responses update **in place**.
- Composer is anchored at the bottom; message history never displaces it.
- Empty state shows the greeting: **"Hi, how can I help you?"**
- Feel: an IDE/operations workspace, not a marketing/chat page.

---

## 4. Right pane — server manager

- Persistent server/connection manager, styled like an infrastructure
  connection manager.
- Header holds an **"Add server"** control.
- Registered servers listed here, groupable/categorizable.
- Server entry shows at minimum: **friendly name**, **host/IP**,
  **bootstrap/connection state**. Never show credentials.
- Pane scrolls independently and remains available while chatting.
- Selecting a server never destroys or navigates away from the active chat.

---

## 5. Add server flow

Two entry points:
- **(A)** User clicks "Add server" in the right pane.
- **(B)** Agent requests the UI action `open_add_server_form` — a frontend UI
  event, not an infrastructure primitive.

```
Agent → open_add_server_form → SSE/UI event → React opens ServerForm modal
  → User enters credentials → POST /api/servers
```

Credentials must never enter: chat messages, chat history, LLM prompts, agent
tool arguments, agent tool results, SSE events, logs, traces.

The modal is **shared** between both entry points.

---

## 6. Server categories

- Right pane supports expandable/collapsible groups.
- Initial implementation uses a **presentation/grouping model** over existing
  server data — no backend category feature unless requested.
- Leave room for future categories: Production, Staging, Development, Database,
  Kubernetes, Other.

---

## 7. Responsive behavior

- **Desktop is primary:** `LEFT | CENTER | RIGHT`.
- Narrower screens collapse panes into overlays/drawers; never shrink all three
  until chat is unusable.
- Space priority: (1) center chat, (2) chat sessions, (3) server manager.
- Collapsed sidebars are always reopenable.

---

## 8. Visual language

Dark, minimal, technical, dense-but-readable, developer/operations oriented,
low visual noise, persistent workspace, clear pane boundaries, restrained
borders, compact controls.

Avoid: large dashboard cards, excessive gradients, marketing hero sections,
oversized buttons, unnecessary animations, excessive rounded containers, generic
SaaS-dashboard look. Aim closer to a developer tool / infrastructure console.

---

## 9. Pane behavior

Independent state per pane:
- **Left:** expanded/collapsed
- **Right:** expanded/collapsed, category open/closed
- **Center:** active session, scroll position, streaming state

Switching sessions or server UI state must not reload the whole application.
State via Zustand, consistent with the existing store design.

---

## 10. Routing

Authenticated workspace is the primary route. No separate dashboard page.

```
/login

/
  ├── session list
  ├── active chat
  └── server manager
```

Routing covers auth and future pages; switching chat sessions does not
navigate to a dashboard.

---

## 11. Component structure

```
AppShell
├── Sidebar / ChatSidebar
│   ├── SessionList
│   └── NewChatButton
├── ChatWorkspace
│   ├── ChatHeader
│   ├── ChatFeed
│   ├── ToolCallBanner
│   └── ChatComposer
└── ServerPanel
    ├── ServerPanelHeader
    ├── AddServerButton
    ├── ServerCategory
    │   └── ServerItem
    └── ServerFormModal
```

Names are guidance; preserve existing project architecture where reasonable.

---

## 12. Frontend milestones (implementation order)

- **F0 — Scaffold:** application shell + three-pane layout.
- **F1 — Auth & workspace routing:** login + authenticated workspace.
- **F2 — Chat/session workspace:** session list, active chat, message
  rendering, composer.
- **F3 — Server manager:** right panel, categories/groups, server state,
  Add Server.
- **F4 — ServerForm:** shared modal + credential-safe submission.
- **F5 — Chat SSE:** streaming chat, tool events, `open_add_server_form` UI
  event.
- **F6 — Polish:** collapse/expand, responsive, loading/error/empty states,
  bootstrap progress, final visual refinement.

---

## 13. Implementation principle

The reference defines spatial layout and interaction model — reproduce its
structure, not the image literally. Production React UI: accessible, keyboard
usable, responsive, componentized, state-driven, consistent with the React +
Vite + TypeScript + Zustand stack. No new UI framework unless the project
already uses one.