# OpenClaw Gateway Chatbot — 后端类设计

> Spring Boot 后端核心类图、接口映射与协作关系。

---

## 1. 系统分层总览

后端分为三个核心层与一组模型/DTO 类：

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    direction TB

    class ChatController {
        -chatService : ChatService
        -sseEmitterManager : SseEmitterManager
        +send(req : ChatSendRequest) SseEmitter
        +abort(req : ChatAbortRequest) ResponseEntity
    }

    class SessionController {
        -sessionService : SessionService
        +listSessions(limit, includeLastMessage) List~SessionInfo~
        +createSession(req : SessionCreateRequest) ResponseEntity
        +getMessages(key, limit) MessageHistory
        +resetSession(key, reason) ResponseEntity
        +deleteSession(key, deleteTranscript) ResponseEntity
    }

    class SseController {
        -sseEmitterManager : SseEmitterManager
        +events() SseEmitter
    }

    class ChatService {
        -gatewayClient : GatewayWebSocketClient
        -sseEmitterManager : SseEmitterManager
        +sendMessage(sessionKey, message) SseEmitter
        +abortMessage(sessionKey, runId) AbortResult
    }

    class SessionService {
        -gatewayClient : GatewayWebSocketClient
        +listSessions(params) List~SessionInfo~
        +createSession(label, message) SessionCreateResult
        +getHistory(sessionKey, limit) MessageHistory
        +resetSession(key, reason) void
        +deleteSession(key, deleteTranscript) void
    }

    class SseEmitterManager {
        -emitters : ConcurrentHashMap~String, SseEmitter~
        -globalEmitters : CopyOnWriteArrayList~SseEmitter~
        +register(runId) SseEmitter
        +emitDelta(runId, text) void
        +emitFinal(runId, message) void
        +emitAborted(runId) void
        +emitError(runId, errorKind, errorMessage) void
        +complete(runId) void
        +broadcastSessionChanged(sessionKey, reason, ts) void
        +registerGlobal() SseEmitter
    }

    class GatewayWebSocketClient {
        -session : WebSocketSession
        -frameCodec : GatewayFrameCodec
        -dispatcher : GatewayEventDispatcher
        -pendingRequests : ConcurrentHashMap~String, CompletableFuture~
        -connected : AtomicBoolean
        +connect() void
        +disconnect() void
        +sendRequest(method, params) CompletableFuture~GatewayResponse~
        +chatSend(sessionKey, message, idempotencyKey) CompletableFuture~ChatSendResult~
        +chatAbort(sessionKey, runId) CompletableFuture~ChatAbortResult~
        +chatHistory(sessionKey, limit) CompletableFuture~MessageHistory~
        +sessionsSubscribe() CompletableFuture~GatewayResponse~
        +sessionsList(params) CompletableFuture~List~SessionInfo~~
        +sessionsCreate(params) CompletableFuture~SessionCreateResult~
        +sessionsReset(key, reason) CompletableFuture~GatewayResponse~
        +sessionsDelete(key, deleteTranscript) CompletableFuture~GatewayResponse~
        -onMessage(frame : String) void
        -onChallenge(nonce, ts) void
        -onEvent(event : GatewayEvent) void
        -reconnect() void
    }

    class GatewayFrameCodec {
        +encodeRequest(id, method, params)$ String
        +encodeConnectParams(token, clientId)$ String
        +decodeResponse(json)$ GatewayResponse
        +decodeEvent(json)$ GatewayEvent
        +extractEventType(json)$ String
    }

    class GatewayEventDispatcher {
        -handlers : Map~String, EventHandler~
        -sseEmitterManager : SseEmitterManager
        -sessionService : SessionService
        +registerHandler(eventType, handler) void
        +dispatch(event : GatewayEvent) void
        +onChatEvent(event : ChatEvent) void
        +onSessionChanged(event : SessionChangedEvent) void
        +onChallenge(nonce, ts) void
        +onShutdown(reason, restartExpectedMs) void
    }

    ChatController --> ChatService : uses
    ChatController --> SseEmitterManager : uses
    SessionController --> SessionService : uses
    SseController --> SseEmitterManager : uses
    ChatService --> GatewayWebSocketClient : uses
    ChatService --> SseEmitterManager : uses
    SessionService --> GatewayWebSocketClient : uses
    GatewayWebSocketClient --> GatewayFrameCodec : uses
    GatewayWebSocketClient --> GatewayEventDispatcher : uses
    GatewayEventDispatcher --> SseEmitterManager : emits SSE
```

**层次职责：**

| 层 | 类 | 职责 |
|----|-----|------|
| **Controller** | `ChatController`, `SessionController`, `SseController` | HTTP 端点、请求校验、响应封装 |
| **Service** | `ChatService`, `SessionService`, `SseEmitterManager` | 业务编排、SSE 生命周期管理 |
| **Gateway Client** | `GatewayWebSocketClient`, `GatewayFrameCodec`, `GatewayEventDispatcher` | WS 协议交互、帧编解码、事件路由 |

---

## 2. Controller 层详细设计

### 2.1 ChatController

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class ChatController {
        <<@RestController>>
        <<@RequestMapping("/api/chat")>>
        -chatService : ChatService
        -sseEmitterManager : SseEmitterManager

        %% POST /api/chat/send → 返回 SSE 流
        +send(req : ChatSendRequest) SseEmitter
        %% POST /api/chat/abort → 中止生成
        +abort(req : ChatAbortRequest) ResponseEntity~AbortResult~
    }

    class ChatSendRequest {
        +sessionKey : String
        +message : String
    }

    class ChatAbortRequest {
        +sessionKey : String
        +runId : String?
    }

    ChatController --> ChatSendRequest : 接收
    ChatController --> ChatAbortRequest : 接收
```

**接口映射：**

| 方法 | HTTP 端点 | 内部调用链 |
|------|----------|-----------|
| `send()` | `POST /api/chat/send` | `chatService.sendMessage()` → 注册 SseEmitter → 返回 SSE 流 |
| `abort()` | `POST /api/chat/abort` | `chatService.abortMessage()` → 同步返回 JSON |

---

### 2.2 SessionController

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class SessionController {
        <<@RestController>>
        <<@RequestMapping("/api/sessions")>>
        -sessionService : SessionService

        +listSessions(limit : int, includeLastMessage : boolean) List~SessionInfo~
        +createSession(req : SessionCreateRequest) ResponseEntity
        +getMessages(key : String, limit : int) MessageHistory
        +resetSession(key : String, body : ResetRequest) ResponseEntity
        +deleteSession(key : String, deleteTranscript : boolean) ResponseEntity
    }

    class SessionCreateRequest {
        +label : String?
        +message : String?
    }

    class ResetRequest {
        +reason : String
    }

    SessionController --> SessionCreateRequest : 接收
    SessionController --> ResetRequest : 接收
```

**接口映射：**

| 方法 | HTTP 端点 | 内部调用链 |
|------|----------|-----------|
| `listSessions()` | `GET /api/sessions` | `sessionService.listSessions()` → `sessions.list` RPC |
| `createSession()` | `POST /api/sessions` | `sessionService.createSession()` → `sessions.create` RPC |
| `getMessages()` | `GET /api/sessions/{key}/messages` | `sessionService.getHistory()` → `chat.history` RPC |
| `resetSession()` | `POST /api/sessions/{key}/reset` | `sessionService.resetSession()` → `sessions.reset` RPC |
| `deleteSession()` | `DELETE /api/sessions/{key}` | `sessionService.deleteSession()` → `sessions.delete` RPC |

---

### 2.3 SseController

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class SseController {
        <<@RestController>>
        -sseEmitterManager : SseEmitterManager

        %% GET /api/events → 全局 SSE 通道（会话变更通知）
        +events() SseEmitter
    }
```

**说明：** `/api/events` 是一个全局 SSE 端点，用于推送 `sessions.changed` 等非 chat 相关的事件。`POST /api/chat/send` 的 SSE 流是独立的（每次请求创建一个），与此全局通道分离。

---

## 3. Service 层详细设计

### 3.1 ChatService

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class ChatService {
        -gatewayClient : GatewayWebSocketClient
        -sseEmitterManager : SseEmitterManager

        +sendMessage(sessionKey : String, message : String) SseEmitter
        +abortMessage(sessionKey : String, runId : String?) AbortResult

        -generateIdempotencyKey() String
        -buildChatSendParams(sessionKey, message, idempotencyKey) Map
    }
```

**`sendMessage()` 内部流程：**

```
1. idempotencyKey = generateIdempotencyKey()     // UUID
2. emitter = sseEmitterManager.register(id)       // 创建 SseEmitter
3. gatewayClient.chatSend(sessionKey, message, id) // 异步发送 WS RPC
4. 等待 ok:true → 返回 emitter                    // SSE 流开始
5. [异步] GatewayEventDispatcher 收到 chat event →
   sseEmitterManager.emitDelta/emitFinal/...       // 推送 SSE
6. [异步] 收到 final/aborted/error →
   sseEmitterManager.complete(id)                  // 关闭 SSE 流
```

### 3.2 SessionService

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class SessionService {
        -gatewayClient : GatewayWebSocketClient

        +listSessions(params : SessionListParams) List~SessionInfo~
        +createSession(label : String?, message : String?) SessionCreateResult
        +getHistory(sessionKey : String, limit : int) MessageHistory
        +resetSession(key : String, reason : String) void
        +deleteSession(key : String, deleteTranscript : boolean) void

        -toSessionInfo(gatewayPayload) SessionInfo
        -toMessageInfo(gatewayPayload) MessageInfo
    }

    class SessionListParams {
        +limit : int
        +activeMinutes : int?
        +includeLastMessage : boolean
        +includeDerivedTitles : boolean
        +search : String?
    }

    SessionService --> SessionListParams : uses
```

**说明：** `SessionService` 的方法都是同步阻塞的（通过 `CompletableFuture.get()` 等待 Gateway RPC 响应）。对于 `listSessions` 和 `getHistory`，直接将 Gateway 返回的 payload 转换为前端需要的 DTO 格式。

### 3.3 SseEmitterManager

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class SseEmitterManager {
        -emitters : ConcurrentHashMap~String, SseEmitter~
        -globalEmitters : CopyOnWriteArrayList~SseEmitter~
        -timeoutMs : long

        %% 为指定 runId 创建并注册 SseEmitter
        +register(runId : String) SseEmitter
        %% 注册全局 SSE 连接（/api/events）
        +registerGlobal() SseEmitter

        %% —— Chat SSE 事件 ——
        +emitDelta(runId : String, text : String) void
        +emitFinal(runId : String, message : Object) void
        +emitAborted(runId : String) void
        +emitError(runId : String, errorKind : String, errorMessage : String) void

        %% —— Session SSE 事件 ——
        +broadcastSessionChanged(sessionKey : String, reason : String, ts : long) void

        %% —— 生命周期 ——
        +complete(runId : String) void
        +remove(runId : String) void

        %% —— SSE 格式 ——
        -buildSseData(event : String, data : Object) String
        -sendKeepalive() void
    }
```

**SSE 事件格式映射：**

| 方法 | SSE event name | 说明 |
|------|---------------|------|
| `emitDelta()` | `chat.delta` | `data: {runId, text}` |
| `emitFinal()` | `chat.final` | `data: {runId, message}` |
| `emitAborted()` | `chat.aborted` | `data: {runId}` |
| `emitError()` | `chat.error` | `data: {runId, errorMessage, errorKind}` |
| `broadcastSessionChanged()` | `sessions.changed` | `data: {sessionKey, reason, ts}` |

---

## 4. Gateway Client 层详细设计

### 4.1 GatewayWebSocketClient

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class GatewayWebSocketClient {
        -wsSession : WebSocketSession
        -frameCodec : GatewayFrameCodec
        -dispatcher : GatewayEventDispatcher
        -pendingRequests : ConcurrentHashMap~String, CompletableFuture~
        -connected : AtomicBoolean
        -requestCounter : AtomicLong
        -gatewayUrl : String
        -gatewayToken : String

        %% —— 连接生命周期 ——
        +afterPropertiesSet() void        @@PostConstruct 启动连接
        +connect() void                    建立 WS 并完成握手
        +disconnect() void                 关闭 WS 连接
        -reconnect() void                  指数退避重连
        -scheduleHeartbeat() void          tick 心跳监控

        %% —— 通用 RPC ——
        +sendRequest(method : String, params : Object) CompletableFuture~GatewayResponse~

        %% —— Chat RPC ——
        +chatSend(sessionKey, message, idempotencyKey) CompletableFuture~ChatSendResult~
        +chatAbort(sessionKey, runId) CompletableFuture~ChatAbortResult~
        +chatHistory(sessionKey, limit) CompletableFuture~MessageHistory~

        %% —— Session RPC ——
        +sessionsSubscribe() CompletableFuture~GatewayResponse~
        +sessionsList(params) CompletableFuture~List~SessionInfo~~
        +sessionsCreate(params) CompletableFuture~SessionCreateResult~
        +sessionsReset(key, reason) CompletableFuture~GatewayResponse~
        +sessionsDelete(key, deleteTranscript) CompletableFuture~GatewayResponse~

        %% —— 内部回调 ——
        -onMessage(frame : String) void
        -onChallenge(nonce : String, ts : long) void
        -onEvent(event : GatewayEvent) void
        -handleResponse(response : GatewayResponse) void
        -nextRequestId() String
    }
```

**RPC 方法与 Gateway 协议的映射：**

| Java 方法 | Gateway RPC method | 参数构建 |
|-----------|-------------------|---------|
| `chatSend()` | `"chat.send"` | `{sessionKey, message, idempotencyKey, deliver:false}` |
| `chatAbort()` | `"chat.abort"` | `{sessionKey, runId?}` |
| `chatHistory()` | `"chat.history"` | `{sessionKey, limit}` |
| `sessionsSubscribe()` | `"sessions.subscribe"` | `{}` |
| `sessionsList()` | `"sessions.list"` | `{limit, includeLastMessage, ...}` |
| `sessionsCreate()` | `"sessions.create"` | `{label?, message?}` |
| `sessionsReset()` | `"sessions.reset"` | `{key, reason}` |
| `sessionsDelete()` | `"sessions.delete"` | `{key, deleteTranscript}` |

**请求-响应匹配机制：**

`sendRequest()` 的工作流程：

```
1. reqId = nextRequestId()                    // 自增 ID
2. future = new CompletableFuture()
3. pendingRequests.put(reqId, future)          // 注册等待
4. json = frameCodec.encodeRequest(reqId, method, params)
5. wsSession.sendMessage(json)                 // 发送到 Gateway
6. return future                               // 调用方通过 .get() 阻塞等待
7. [异步] onMessage() 收到 res → handleResponse()
   → pendingRequests.remove(reqId)
   → future.complete(response)                 // 解除阻塞
```

---

### 4.2 GatewayFrameCodec

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class GatewayFrameCodec {
        <<utility>>
        -objectMapper : ObjectMapper$

        %% —— 编码（Java → JSON）——
        +encodeRequest(id : String, method : String, params : Object)$ String
        +encodeConnectParams(token : String, clientId : String)$ String
        +encodeSubscribeRequest(id : String)$ String

        %% —— 解码（JSON → Java）——
        +decodeResponse(json : String)$ GatewayResponse
        +decodeEvent(json : String)$ GatewayEvent
        +extractEventType(json : String)$ String

        %% —— 内部辅助 ——
        -buildFrame(type : String, fields : Map)$ String
        -parseJson(json : String)$ Map
    }

    class GatewayResponse {
        +id : String
        +ok : boolean
        +payload : Object?
        +error : ErrorShape?
    }

    class GatewayEvent {
        +event : String
        +payload : Object
        +seq : long
    }

    class ErrorShape {
        +code : String
        +message : String
        +details : Object?
        +retryable : boolean?
        +retryAfterMs : long?
    }

    GatewayFrameCodec ..> GatewayResponse : creates
    GatewayFrameCodec ..> GatewayEvent : creates
    GatewayResponse --> ErrorShape : contains
```

**说明：** `GatewayFrameCodec` 是无状态工具类，所有公开方法为 `static`。负责三种 Gateway 帧格式的序列化/反序列化：

| 帧类型 | 编码方法 | 解码方法 |
|--------|---------|---------|
| Request (`"req"`) | `encodeRequest()` | — |
| Response (`"res"`) | — | `decodeResponse()` |
| Event (`"event"`) | — | `decodeEvent()` |

---

### 4.3 GatewayEventDispatcher

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    class GatewayEventDispatcher {
        -sseEmitterManager : SseEmitterManager
        -chatEventHandlers : List~ChatEventHandler~
        -sessionChangeHandlers : List~SessionChangeHandler~

        +setSseEmitterManager(manager : SseEmitterManager) void
        +addChatEventHandler(handler : ChatEventHandler) void
        +addSessionChangeHandler(handler : SessionChangeHandler) void

        %% 主入口：由 GatewayWebSocketClient.onMessage() 调用
        +dispatch(event : GatewayEvent) void

        %% 内部分发
        -onChatEvent(payload : Map) void
        -onSessionChanged(payload : Map) void
        -onTick(ts : long) void
        -onShutdown(payload : Map) void
    }

    class ChatEventHandler {
        <<interface>>
        +onDelta(runId : String, text : String) void
        +onFinal(runId : String, message : Object) void
        +onAborted(runId : String) void
        +onError(runId : String, errorKind : String, errorMessage : String) void
    }

    class SessionChangeHandler {
        <<interface>>
        +onSessionChanged(sessionKey : String, reason : String, ts : long) void
    }

    GatewayEventDispatcher --> ChatEventHandler : notifies
    GatewayEventDispatcher --> SessionChangeHandler : notifies
```

**事件路由逻辑：**

```
GatewayWebSocketClient.onMessage(json)
  → GatewayFrameCodec.extractEventType(json)
  → switch(eventType)
      case "chat"            → onChatEvent(payload)
                                 switch(state)
                                   "delta"   → sseEmitterManager.emitDelta()
                                   "final"   → sseEmitterManager.emitFinal() + complete()
                                   "aborted" → sseEmitterManager.emitAborted() + complete()
                                   "error"   → sseEmitterManager.emitError() + complete()
      case "sessions.changed" → onSessionChanged(payload)
                                  → sseEmitterManager.broadcastSessionChanged()
      case "tick"            → scheduleHeartbeat() // 重置心跳超时
      case "shutdown"        → onShutdown(payload) // 触发重连
```

---

## 5. 模型 / DTO 类

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'classBorder': '#61afef', 'classBkg': '#282c34', 'classTextColor': '#abb2bf', 'classAttributeTextColor': '#abb2bf' }}}%%
classDiagram
    direction LR

    class SessionInfo {
        +key : String
        +label : String?
        +lastMessage : String?
        +updatedAt : String
    }

    class MessageInfo {
        +role : String
        +content : String
        +timestamp : String
    }

    class MessageHistory {
        +sessionKey : String
        +messages : List~MessageInfo~
    }

    class ChatSendResult {
        +runId : String
        +status : String
    }

    class ChatAbortResult {
        +aborted : boolean
        +runIds : List~String~
    }

    class SessionCreateResult {
        +sessionKey : String
        +sessionId : String
    }

    class AbortResult {
        +aborted : boolean
        +runIds : List~String~
    }

    MessageHistory --> MessageInfo : contains
```

---

## 6. 核心协作流程

### 6.1 应用启动与 WebSocket 连接建立（挑战握手 + 设备签名连接）

本节从**后端类协作**角度总结 Gateway 的建连流程。对新的 operator 客户端，推荐把握手建模为：**WebSocket 建立后，按正常 operator 路径处理 `connect.challenge`，再发送带共享认证信息与 `device` 签名的 `connect` 请求；首次连接成功后持久化 `hello-ok.auth.deviceToken`，后续重连优先复用同一设备身份与已持久化的 device token。**

如果部署侧仍使用共享 token（例如通过环境变量 `OPENCLAW_GATEWAY_TOKEN` 注入），它属于**首次连接时的共享认证材料**，而不是“可替代设备身份的主流程”。在当前客户端实现里，token-only 更适合作为 plain HTTP / insecure compatibility 等受限场景下的回退路径，不应作为新后端客户端的默认建模方式。

**认证流程概览：**

```
WebSocket open → Gateway 按正常 operator 路径发 `connect.challenge` → Chatbot 使用共享认证信息 + `device` 签名发送 connect → Gateway 返回 hello-ok（含 deviceToken）→ Chatbot 持久化 deviceToken → 再订阅 sessions / 进入后续 RPC
```

按这一设计建模的原因：
1. **符合协议主路径**：`connect.challenge` 是新的 operator 客户端正常握手的一部分，不应被当成可忽略事件
2. **稳定承载设备身份与 scopes**：首次连接使用共享认证信息 + `device` 签名，更符合当前 Gateway 对 operator 客户端的建模
3. **建立可持续的重连路径**：首次成功后可持久化 `hello-ok.auth.deviceToken`，后续重连优先复用已批准的 device token
4. **保留兼容回退说明**：token-only 可能在不安全上下文或兼容模式下出现，但不应在类设计中被写成推荐主流程

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'actorBkg': '#282c34', 'actorBorder': '#61afef', 'actorTextColor': '#abb2bf', 'signalColor': '#abb2bf', 'signalTextColor': '#abb2bf', 'noteBkgColor': '#21252b', 'noteTextColor': '#abb2bf', 'noteBorderColor': '#61afef'}}}%%
sequenceDiagram
    autonumber
    participant App as Spring Boot App
    participant GWC as GatewayWebSocketClient
    participant FC as GatewayFrameCodec
    participant GW as Gateway (WebSocket)

    Note over App,GW: 首次连接通常使用共享认证信息 + device-signed connect<br/>若部署使用共享 token，可来自环境变量 OPENCLAW_GATEWAY_TOKEN

    Note over App: @PostConstruct 触发

    App->>GWC: afterPropertiesSet()
    GWC->>GWC: connect()
    Note over GWC: 读取共享认证材料，准备本地 device identity / 已持久化 deviceToken

    GWC->>GW: WS open
    GW-->>GWC: event "connect.challenge" {nonce, ts}

    Note over GWC,FC: 准备 ConnectParams：<br/>minProtocol=3, maxProtocol=3<br/>client.id="webchat", client.mode="backend"<br/>role="operator"<br/>scopes=["operator.read", "operator.write"]<br/>auth: 可包含 token/password，且在当前实现中也可能带已解析的 deviceToken<br/>device: {id, publicKey, signature, signedAt, nonce}

    GWC->>FC: encodeConnectParams(...)
    FC-->>GWC: JSON connect frame

    GWC->>GW: req "connect" {<br/>  minProtocol:3, maxProtocol:3,<br/>  client:{id:"webchat", mode:"backend"},<br/>  auth:{token?, password?, deviceToken?},<br/>  role:"operator",<br/>  scopes:["operator.read","operator.write"],<br/>  device:{id, publicKey, signature, signedAt, nonce}<br/>}

    Note over GW: 按当前认证路径校验 auth 字段，<br/>并校验 challenge nonce 对应的 device 签名

    GW-->>GWC: res {ok:true, payload: hello-ok}

    Note over GWC: hello-ok 关键字段：<br/>- auth.deviceToken：首次成功后需持久化<br/>- auth.role / auth.scopes<br/>- features.methods / features.events<br/>- policy / snapshot

    GWC->>GWC: handleResponse() → connected.set(true)
    GWC->>GWC: persist hello-ok.auth.deviceToken

    GWC->>GWC: sessionsSubscribe()
    GWC->>FC: encodeRequest(id, "sessions.subscribe", {})
    GWC->>GW: req "sessions.subscribe" {}
    GW-->>GWC: res {ok:true, subscribed:true}

    GWC->>GWC: scheduleHeartbeat()
    Note over GWC,GW: 后续重连通常仍按当前 operator 路径处理 connect.challenge，<br/>优先复用已持久化的 deviceToken + 同一 device identity
```

**关键点：**

| 要点 | 说明 |
|------|------|
| **共享认证材料来源** | 若部署使用共享 token，可由环境变量 `OPENCLAW_GATEWAY_TOKEN` 注入；它主要用于首次连接时的共享认证 |
| **典型握手顺序** | 正常 operator 路径通常表现为 `WS open` → `connect.challenge` → device-signed `connect` → `hello-ok` → 持久化 `auth.deviceToken` → 再发 `sessions.subscribe` 等后续 RPC |
| **首次连接认证方式** | 推荐使用共享 `token/password` + `device` 签名；在当前实现中，`auth` 里也可能携带已解析的 `deviceToken` |
| **重连认证方式** | 首次成功后持久化 `hello-ok.auth.deviceToken`，后续重连优先复用该 token，并保持同一 device identity |
| **connect.challenge** | challenge 中的 nonce 需要进入 `device` 签名 payload；对新的 operator 客户端，应把它视为正常握手路径的一部分 |
| **token-only 的定位** | 在当前客户端实现里，可作为 plain HTTP / insecure compatibility 等场景下的 fallback，但不应写成推荐主流程 |
| **连接生命周期** | `GatewayWebSocketClient` 在 `@PostConstruct` 阶段自动建连；收到 `hello-ok` 后再注册 `sessions.subscribe()` 并启动心跳监控 |
| **心跳监控** | `scheduleHeartbeat()` 监控 `tick` 事件，超时则触发 `reconnect()`；重连流程仍应重复 challenge + connect 链路 |

---

### 6.2 用户发送消息（HTTP → WS → SSE 全链路）

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'actorBkg': '#282c34', 'actorBorder': '#61afef', 'actorTextColor': '#abb2bf', 'signalColor': '#abb2bf', 'signalTextColor': '#abb2bf', 'noteBkgColor': '#21252b', 'noteTextColor': '#abb2bf', 'noteBorderColor': '#61afef'}}}%%
sequenceDiagram
    autonumber
    participant F as React (Frontend)
    participant CC as ChatController
    participant CS as ChatService
    participant SEM as SseEmitterManager
    participant GWC as GatewayWebSocketClient
    participant FC as GatewayFrameCodec
    participant ED as GatewayEventDispatcher
    participant GW as Gateway

    F->>CC: POST /api/chat/send {sessionKey, message}
    CC->>CS: sendMessage(sessionKey, message)
    CS->>CS: generateIdempotencyKey() → UUID
    CS->>SEM: register(idempotencyKey) → SseEmitter
    CS->>GWC: chatSend(sessionKey, message, idempotencyKey)
    GWC->>FC: encodeRequest(id, "chat.send", {sessionKey, message, idempotencyKey, deliver:false})
    FC-->>GWC: JSON request frame
    GWC->>GW: req "chat.send" {…}
    GW-->>GWC: res {ok:true, runId:"run-abc"}

    GWC->>GWC: handleResponse() → future.complete()
    CS-->>SEM: [关联 runId 到 emitter]
    CC-->>F: HTTP 200 SSE stream (SseEmitter)

    Note over GW: Agent 开始推理...

    loop 流式输出 (~150ms 间隔)
        GW-->>GWC: event "chat" {runId, state:"delta", message:{content:"累积文本"}}
        GWC->>FC: decodeEvent(json) → GatewayEvent
        GWC->>ED: dispatch(event)
        ED->>ED: onChatEvent(payload) → state="delta"
        ED->>SEM: emitDelta(runId, text)
        SEM-->>F: SSE event: chat.delta<br/>data: {runId, text}
    end

    GW-->>GWC: event "chat" {runId, state:"final", message:{…}}
    GWC->>ED: dispatch(event)
    ED->>ED: onChatEvent(payload) → state="final"
    ED->>SEM: emitFinal(runId, message)
    SEM-->>F: SSE event: chat.final<br/>data: {runId, message}
    ED->>SEM: complete(runId)
    Note over SEM,F: SSE 流关闭
```

---

### 6.3 会话管理（以"创建新会话"为例）

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'actorBkg': '#282c34', 'actorBorder': '#61afef', 'actorTextColor': '#abb2bf', 'signalColor': '#abb2bf', 'signalTextColor': '#abb2bf', 'noteBkgColor': '#21252b', 'noteTextColor': '#abb2bf', 'noteBorderColor': '#61afef'}}}%%
sequenceDiagram
    autonumber
    participant F as React (Frontend)
    participant SC as SessionController
    participant SS as SessionService
    participant GWC as GatewayWebSocketClient
    participant FC as GatewayFrameCodec
    participant GW as Gateway

    F->>SC: POST /api/sessions {label:"My Chat"}
    SC->>SS: createSession(label, null)
    SS->>GWC: sessionsCreate({label:"My Chat"})
    GWC->>FC: encodeRequest(id, "sessions.create", {label:"My Chat"})
    GWC->>GW: req "sessions.create" {label:"My Chat"}
    GW-->>GWC: res {ok:true, key:"generated-key", sessionId:"…"}

    GWC->>GWC: handleResponse() → future.complete()

    Note over GW: sessions.changed 事件自动推送

    GW-->>GWC: event "sessions.changed" {sessionKey, reason:"create"}
    GWC->>GWC: [ED.dispatch → SEM.broadcastSessionChanged()]

    SS-->>SC: SessionCreateResult {sessionKey, sessionId}
    SC-->>F: HTTP 201 {sessionKey:"generated-key"}
```

---

### 6.4 会话变更通知（自动推送）

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#282c34', 'edgeLabelBackground': '#282c34', 'mainBkg': '#282c34', 'primaryTextColor': '#abb2bf', 'lineColor': '#abb2bf', 'tertiaryColor': '#21252b', 'clusterBkg': '#21252b', 'clusterBorder': '#61afef', 'fontFamily': 'arial', 'actorBkg': '#282c34', 'actorBorder': '#61afef', 'actorTextColor': '#abb2bf', 'signalColor': '#abb2bf', 'signalTextColor': '#abb2bf', 'noteBkgColor': '#21252b', 'noteTextColor': '#abb2bf', 'noteBorderColor': '#61afef'}}}%%
sequenceDiagram
    autonumber
    participant GW as Gateway
    participant GWC as GatewayWebSocketClient
    participant ED as GatewayEventDispatcher
    participant SEM as SseEmitterManager
    participant F as React (Frontend)

    Note over GW: 任何会话被创建/删除/修改

    GW-->>GWC: event "sessions.changed" {sessionKey, reason, ts}
    GWC->>ED: dispatch(event)
    ED->>ED: onSessionChanged(payload)
    ED->>SEM: broadcastSessionChanged(sessionKey, reason, ts)
    SEM-->>F: SSE event: sessions.changed<br/>data: {sessionKey, reason, ts}

    Note over F: 前端根据 reason 更新侧边栏
```

---

## 7. 设计决策总结

| 决策 | 说明 |
|------|------|
| **GatewayWebSocketClient 单例** | 单用户独占场景，全局只需一条持久 WS 连接。通过 `@Component` 自动管理 |
| **SseEmitterManager 管理生命周期** | `register(runId)` 创建 emitter，收到 final/aborted/error 后自动 `complete()`。超时由 Spring `SseEmitter` 默认超时机制兜底 |
| **GatewayFrameCodec 无状态** | 纯编解码工具类，所有方法为 static，不持有任何状态 |
| **GatewayEventDispatcher 回调驱动** | 通过 `ChatEventHandler` / `SessionChangeHandler` 接口解耦，方便测试和扩展 |
| **请求-响应异步匹配** | `pendingRequests` Map 以 request ID 为 key，`CompletableFuture` 实现 RPC 的请求-响应关联 |
| **delta 文本替换而非追加** | Gateway 协议保证 delta 包含完整累积文本，后端直接转发给前端替换显示 |
