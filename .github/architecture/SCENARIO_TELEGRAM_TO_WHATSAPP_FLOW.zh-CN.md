# 场景拆解：A( Telegram ) 发给 B( WhatsApp ) 的完整通信流程

> 目标场景：A 与 B 都是 OpenClaw 中的 Agent。A 侧外部渠道是 Telegram，B 侧外部渠道是 WhatsApp。希望实现：A 收到消息后，B 能收到并回复。

---

## 1. 边界与前提

### 1.1 边界

- 本仓库（ClawSwarm）负责：**跨 Agent 编排、消息分发、回调状态机、会话通知**。
- 本仓库不直接实现：Telegram/WhatsApp 协议对接（这部分在 OpenClaw 及其渠道适配器域内）。

### 1.2 前提条件

1. A、B 都已注册为 ClawSwarm 的 `AgentProfile`。
2. A、B 所在 OpenClaw 实例已在 ClawSwarm 中配置为 `OpenClawInstance`。
3. 目标实例已安装并启用 `clawswarm` channel 插件。
4. 会话模式满足路由条件（direct 指定 B，或 group 中 mention 到 B）。

---

## 2. 端到端全流程（分层）

## 阶段 A：Agent 渠道入站（OpenClaw 域）

1. 用户在 Telegram 向 A 发消息。
2. OpenClaw 渠道适配器将该消息规范化为统一入站上下文（chatId、from、text 等）。
3. ClawSwarm 插件最终接收的是统一 `InboundMessage` 结构，而非 Telegram 专有协议。

> 说明：该阶段属于 OpenClaw 渠道层，不在 ClawSwarm 仓库内直接实现。

---

## 阶段 B：Swarm 写入消息并发起分发（scheduler-server）

4. 上层入口调用 `POST /api/conversations/{conversation_id}/messages`。
5. `scheduler-server` 先创建并持久化一条 `Message(status=pending)`。
6. 根据会话类型分流：
   - direct -> `dispatch_direct_message`
   - group -> `dispatch_group_message`
7. 分发函数为每个目标 Agent 创建 `MessageDispatch(status=pending)`。
8. group 模式下会按 `instance_id` 分桶，支持跨实例同时投递（A/B 可不在同一实例）。

---

## 阶段 C：Swarm -> 插件（签名 inbound）

9. `scheduler-server` 通过 `ChannelClient.send_inbound()` 向目标实例插件发送：
   - `POST /clawswarm/v1/inbound`
   - 带签名头（accountId/timestamp/nonce/signature）
10. inbound payload 中包含：
   - messageId/accountId
   - chat(type/chatId/groupId)
   - from
   - text
   - directAgentId 或 targetAgentIds/mentions

---

## 阶段 D：插件路由与执行（channel）

11. 插件入口 `handleInboundRoute` 执行校验链：
   - body 大小限制
   - HMAC 验签
   - JSON 解析
   - Schema 校验
12. 插件做路由决策 `resolveRoute`：
   - `DIRECT`
   - `GROUP_MENTION`
   - `GROUP_BROADCAST`
13. 插件先返回 ACK（accepted + traceId），随后异步执行。
14. 异步调度 `runInboundDispatch`：
   - DIRECT -> `dispatchDirect`
   - GROUP_* -> `dispatchGroup`（逐 Agent 排队并复用 dispatchDirect）
15. `dispatchDirect` 执行顺序：
   - prepare（幂等 + sessionKey）
   - 回推 `run.accepted`
   - 调用 OpenClaw runtime 跑 Agent
   - 流式回推 `reply.chunk`
   - 结束回推 `reply.final`（异常则 `run.error`）

---

## 阶段 E：插件 -> Swarm（回调事件）

16. 插件回调客户端 `HttpClawSwarmCallbackClient` 发送：
   - `POST /api/v1/clawswarm/events`
   - Bearer token + HMAC(timestamp.body)
   - 事件类型：accepted/chunk/final/error
17. `scheduler-server` 在 callbacks 路由校验 token 与可选签名后，交给 `handle_callback_event`。
18. `handle_callback_event` 处理：
   - 基于 messageId + instance + agent 定位 `MessageDispatch`
   - 去重写入 `MessageCallbackEvent`
   - 推进 dispatch/message 状态机
   - chunk 时累计 agent 消息内容
   - final 时写入/完成 agent 消息

---

## 阶段 F：Swarm 通知与会话刷新（前端）

19. scheduler 在“发送消息”和“回调更新”后均推送 `conversation.updated`。
20. 前端 `useConversationTransport`：
   - 优先 WebSocket 收通知
   - 收到后调用 HTTP 增量拉取
   - WS 不可用则回退 polling

---

## 3. 数据对象状态演进（核心）

### Message
- 初始：`pending`
- 接收到 chunk：`streaming`
- 接收到 final：`completed`
- 执行失败：`failed`

### MessageDispatch
- 初始：`pending`
- accepted：`accepted`
- chunk：`streaming`
- final：`completed`
- error：`failed`

### MessageCallbackEvent
- 每个回调事件落一条（支持 eventId 去重），用于可追踪审计与排障。

---

## 4. 从“业务语义”看这个场景

- **A 从 Telegram 收到消息**：发生在 OpenClaw 的渠道接入层。
- **A -> B 的跨 Agent 通信**：通过 ClawSwarm 的路由分发和插件执行闭环实现。
- **B 在 WhatsApp 回出**：发生在 OpenClaw 的渠道出站层。

因此，ClawSwarm 在该场景中的关键职责是：

1. 统一消息模型与路由策略（direct/group/mention）。
2. 保证跨实例、跨 Agent 的可靠分发。
3. 通过 callback 把执行过程（accepted/chunk/final/error）回写为可观测状态。
4. 对前端提供实时通知 + 增量一致性拉取。

---

## 5. 关键文件索引

### scheduler-server
- `src/api/routes/conversations.py`
- `src/services/conversation_dispatch_service.py`
- `src/integrations/channel_client.py`
- `src/api/routes/callbacks.py`
- `src/services/callback_event_service.py`
- `src/services/conversation_events.py`
- `src/api/routes/ws.py`

### channel
- `src/http/inbound.ts`
- `src/core/routing/resolveRoute.ts`
- `src/flows/inbound/inboundDispatch.ts`
- `src/flows/dispatch/dispatchDirect.ts`
- `src/flows/dispatch/dispatchGroup.ts`
- `src/flows/dispatch/directRun.ts`
- `src/flows/dispatch/directPublish.ts`
- `src/flows/callback/client.ts`
- `src/openclaw/runtime/pluginRuntimeAdapter.ts`
- `src/openclaw/runtime/chatGateway.ts`

### web-client
- `src/composables/useConversationTransport.ts`

