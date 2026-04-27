# ClawSwarm Repository Function Analysis & Software Architecture (Detailed)

## 1) What this repository implements

ClawSwarm is a multi-agent orchestration system centered around collaborative conversations.

It has three main subsystems:

- `scheduler-server`: central orchestration backend (FastAPI + SQLAlchemy), managing instances, agents, conversations, messages, tasks, projects/documents, and callback state.
- `channel`: OpenClaw channel plugin that bridges scheduler requests to OpenClaw runtime and sends execution callbacks back.
- `web-client`: Vue-based UI for humans to manage instances/agents and operate conversations.

Primary conversation modes:

1. direct chat (human ↔ one agent)
2. group chat (human ↔ multiple agents with mention routing)
3. agent-dialogue (agent ↔ agent relay)

---

## 2) Top-level architecture (deployment/runtime)

```mermaid
flowchart TB
    U[Human User / Browser] --> W[web-client\nVue3 + Pinia + Router]
    W -->|REST + WebSocket| S[scheduler-server\nFastAPI]

    S -->|Inbound webhook| C[channel plugin\nOpenClaw clawswarm channel]
    C -->|Runtime Adapter| O[OpenClaw Gateway/Runtime]
    O --> A1[Agent A]
    O --> A2[Agent B]
    O --> A3[Agent N]

    C -->|Callback events\nrun.accepted/reply.chunk/reply.final| S

    S --> DB[(SQLite\nconversations/messages/dispatch/tasks/projects)]
```

---

## 3) scheduler-server layered architecture

```mermaid
flowchart LR
    subgraph API[API Layer - src/api/routes]
      R1[auth/instances/agents]
      R2[conversations/groups]
      R3[agent_dialogues/callbacks/ws]
      R4[tasks/projects/address_book]
    end

    subgraph SVC[Service Layer - src/services]
      S1[conversation_dispatch_service]
      S2[callback_event_service]
      S3[agent_dialogue_runner/state/context]
      S4[project_service + document_service]
      S5[conversation_query + event hub]
    end

    subgraph DOM[Domain Model - src/models]
      M1[Conversation]
      M2[Message]
      M3[MessageDispatch + CallbackEvent]
      M4[OpenClawInstance + AgentProfile]
      M5[Task + TaskEvent]
      M6[Project + ProjectDocument]
      M7[ChatGroup + ChatGroupMember]
      M8[AgentDialogue]
    end

    subgraph INFRA[Infra Layer]
      I1[channel_client HTTP]
      I2[core/db + config + security]
      I3[SQLite]
    end

    API --> SVC --> DOM --> INFRA
    SVC --> I1
    DOM --> I3
```

Key backend responsibilities:

- `main.py`: app assembly, schema bootstrap, auth middleware, and optional static hosting for `web-client` build output.
- `conversations + dispatch service`: persist user message first, then dispatch to direct/group routing.
- `callbacks + callback service`: consume plugin callbacks and advance dispatch/message states (`accepted/streaming/completed/failed`).
- `agent-dialogues`: create/pause/resume/stop and turn relay logic for agent-to-agent conversations.

---

## 4) Channel plugin architecture (OpenClaw bridge)

```mermaid
flowchart LR
    subgraph ENTRY[Plugin Entry]
      P1[app/plugin.ts]
      P2[app/runtime.ts]
    end

    subgraph HTTP[HTTP Routes]
      H1[catalog/admin routes]
      H2[inbound webhook route]
    end

    subgraph ROUTE[Routing Core]
      C1[resolveRoute DIRECT/GROUP_MENTION/GROUP_BROADCAST]
      C2[mentions/sessionKey/idempotency]
    end

    subgraph FLOWS[Dispatch Flows]
      F1[dispatchDirect]
      F2[dispatchGroup]
      F3[group queue + state]
      F4[outbound sendText for agent_dialogue.start]
    end

    subgraph RUNTIME[OpenClaw Runtime Adapter]
      O1[plugin_runtime]
      O2[chat_completions]
      O3[transport auto switch]
    end

    subgraph CALLBACK[Callback Client]
      CB1[POST /api/v1/clawswarm/events]
      CB2[retry + timeout]
    end

    ENTRY --> HTTP --> ROUTE --> FLOWS --> RUNTIME
    FLOWS --> CALLBACK
```

Core plugin behavior:

- Inbound webhook flow: verify signature → validate payload → resolve route → ACK quickly → async dispatch.
- Route policy:
  - direct: resolve exactly one target agent
  - group mention: route only to mentioned agents
  - group broadcast: route to request/default target set with upper bounds
- Dispatch policy:
  - `dispatchDirect`: single-agent execution + callback emission
  - `dispatchGroup`: multi-agent queueing and aggregated state

---

## 5) web-client architecture

```mermaid
flowchart TB
    subgraph APP[App Shell]
      A1[main.ts]
      A2[router/index.ts]
      A3[AppRoot/MainLayout]
    end

    subgraph PAGES[Feature Pages]
      P1[Messages]
      P2[OpenClaws Instances]
      P3[Projects + Documents]
      P4[Tasks]
      P5[Settings + Login]
    end

    subgraph STATE[Stores / Composables]
      S1[auth/store]
      S2[conversation/group/addressBook]
      S3[openclaw/project/task]
      S4[useConversationTransport\nWebSocket + Polling fallback]
    end

    subgraph API[API Clients]
      C1[api/*.ts]
      C2[REST to scheduler]
      C3[WS /ws/conversations/:id]
    end

    APP --> PAGES --> STATE --> API
```

---

## 6) Core sequence flows

### 6.1 User sends direct/group message

```mermaid
sequenceDiagram
    participant UI as web-client
    participant S as scheduler-server
    participant DB as SQLite
    participant C as channel plugin
    participant O as OpenClaw runtime

    UI->>S: POST /api/conversations/{id}/messages
    S->>DB: insert Message(status=pending)
    S->>S: dispatch_direct/group
    S->>C: POST /clawswarm/v1/inbound
    C-->>S: 200 accepted(traceId)
    C->>O: run agent turn (async)
    O-->>C: chunks/final
    C->>S: POST /api/v1/clawswarm/events
    S->>DB: update dispatch/message status
    S-->>UI: WS event notify
    UI->>S: GET messages incremental
```

### 6.2 Agent-dialogue relay

```mermaid
sequenceDiagram
    participant A as Source Agent (via plugin send-text)
    participant S as scheduler-server
    participant C as channel plugin
    participant B as Target Agent

    A->>S: /api/v1/clawswarm/send-text kind=agent_dialogue.start
    S->>S: create/reuse AgentDialogue + Conversation
    S->>C: dispatch opening turn
    C->>B: run text turn
    B-->>C: reply.final
    C->>S: callback event
    S->>S: continue dialogue
    S->>C: relay to next agent
```

---

## 7) Simplified data-domain ER

```mermaid
erDiagram
    OPENCLAW_INSTANCES ||--o{ AGENT_PROFILES : hosts
    CHAT_GROUPS ||--o{ CHAT_GROUP_MEMBERS : contains
    AGENT_PROFILES ||--o{ CHAT_GROUP_MEMBERS : joins

    CONVERSATIONS ||--o{ MESSAGES : has
    CONVERSATIONS ||--o{ MESSAGE_DISPATCHES : has
    MESSAGE_DISPATCHES ||--o{ MESSAGE_CALLBACK_EVENTS : records

    AGENT_DIALOGUES ||--|| CONVERSATIONS : binds

    PROJECTS ||--o{ PROJECT_DOCUMENTS : has

    TASKS ||--o{ TASK_EVENTS : timeline
    TASKS ||--o{ TASKS : children
```

---

## 8) Functional boundaries summary

1. Unified orchestration across direct/group/agent-dialogue.
2. Plugin-mediated execution bridge with callback-driven state updates.
3. Observable pipeline with message/dispatch/callback-event tracking.
4. Operational UI and APIs for instances, groups, projects, and tasks.
5. Real-time updates via WebSocket notify + incremental pull fallback.
