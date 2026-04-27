# ClawSwarm 仓库功能分析与软件结构图（详细版）

## 1. 仓库整体定位

ClawSwarm 是一个围绕 **多 Agent 协作会话** 设计的系统，核心由三部分组成：

- `scheduler-server`：调度中枢（FastAPI + SQLAlchemy），负责实例、Agent、会话、消息、任务、项目文档、回调状态。
- `channel`：OpenClaw 插件，负责把调度中心消息投递到 OpenClaw Agent，并把执行回调写回调度中心。
- `web-client`：面向人类用户（老板/管理员）的管理与会话 UI。

其核心目标是把传统单 agent 对话扩展为：

1) 单聊（human ↔ agent）
2) 群聊（human ↔ 多 agent，支持 @mention 路由）
3) agent-dialogue（agent ↔ agent 自动接力）

---

## 2. 顶层软件结构图（部署 + 运行视角）

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

## 3. scheduler-server 分层结构图（后端）

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

### 后端关键职责

- `main.py` 统一挂载 API 路由、初始化表结构、默认用户、鉴权中间件、并可直接托管前端静态资源。  
- `conversations.py + conversation_dispatch_service.py` 负责“先落消息，再分发”的核心写路径（direct/group）。  
- `callbacks.py + callback_event_service.py` 负责把插件回调映射成 dispatch/message 状态机，支持 chunk/final/error。  
- `agent_dialogues` 路径负责 agent-to-agent 的生命周期（创建、暂停、恢复、停止、插话）。

---

## 4. channel 插件结构图（OpenClaw 侧桥接层）

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

### 插件核心逻辑

- 入站 `/clawswarm/v1/inbound`：验签 → JSON 校验 → 路由决策 → 先 ACK → 后台异步调度。
- 路由策略：
  - direct：必须定位单个 agent；
  - group mention：仅投递给被 @ 的 agent；
  - group broadcast：投递给目标列表/默认列表并受上限限制。
- 调度策略：
  - `dispatchDirect` 负责单 agent 的 accepted/chunk/final/error 回调闭环；
  - `dispatchGroup` 负责多 agent 并发排队并聚合状态。

---

## 5. web-client 结构图（前端）

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

### 前端核心特征

- 路由上明确区分公开页（登录）和主工作台页面。
- 会话传输层优先 WebSocket；失败后自动回退轮询，保证弱网环境仍可用。
- 功能面覆盖：消息、群聊、OpenClaw 实例、任务、项目文档。

---

## 6. 核心业务序列图（消息闭环）

### 6.1 用户发起 direct/group 消息

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

### 6.2 agent-dialogue（agent ↔ agent）

```mermaid
sequenceDiagram
    participant A as Source Agent (via plugin send-text)
    participant S as scheduler-server
    participant C as channel plugin
    participant B as Target Agent

    A->>S: /api/v1/clawswarm/send-text kind=agent_dialogue.start
    S->>S: create/reuse AgentDialogue + Conversation
    S->>C: dispatch opening turn to target/source agent
    C->>B: run text turn
    B-->>C: reply.final
    C->>S: callback event
    S->>S: continue_agent_dialogue_after_reply
    S->>C: relay to next agent
```

---

## 7. 数据域（简化 ER）

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

## 8. 功能边界总结

ClawSwarm 当前实现的能力可归纳为：

1. **多会话类型统一编排**：direct/group/agent_dialogue 共用消息与分发表。
2. **插件化执行桥接**：调度中心不直接跑 agent，而通过 OpenClaw channel 插件执行并回调。
3. **可观测状态机**：message + dispatch + callback_event 形成可追踪闭环。
4. **管理面完整度较高**：实例、Agent、群组、项目文档、任务等均有 API/UI。
5. **实时体验渐进增强**：WebSocket 通知 + HTTP 增量拉取 + polling fallback。

