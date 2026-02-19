# AionUi Personal Assistant Feature Development Plan

> This document records the complete development plan for the personal assistant feature, including architecture design, plugin system, and interaction design.

---

## 1. Feature Overview

### 1.1 Basic Information

- **Feature Name**: Personal Assistant Feature
- **Module**: Agent Layer, Conversation System
- **Involved Processes**: Main Process, Worker
- **Runtime Environment**: GUI mode (AionUi running)

### 1.2 Feature Description

1. Similar to WebUI functionality, users can directly use Aion features through their personal terminal
2. Primarily involves personal user's IM communication tools (Telegram, Lark/Feishu, etc.)
3. Build a 7×24 hour personal terminal assistant
4. **Implemented Platforms**: Telegram (grammY), Lark/Feishu (official SDK)
5. **Supported Agents**: Gemini, ACP, Codex

### 1.3 User Scenarios

```
Trigger: User sends a message via mobile IM tool (e.g., Telegram)
Process: Platform bot receives message → forwards to Aion Agent → LLM processes
Result: After processing, push result back to user via the same platform
```

### 1.4 Reference Projects

- **Clawdbot**: https://github.com/clawdbot/clawdbot
- Adopted its plugin design, pairing security mode, Channel abstraction and other design concepts

---

## 2. Overall Architecture

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  ChannelManager (Singleton)                  │
│                  (Unified management of all components)      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │PluginManager│ │SessionManager│ │PairingService│           │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
│  ┌─────────────┐ ┌─────────────────────────────┐            │
│  │ActionExecutor│ │ChannelMessageService         │            │
│  └─────────────┘ └─────────────────────────────┘            │
└────────────────────┼────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 1: Plugin                          │
│                     (Platform Adapter Layer)                 │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐ ┌──────────┐                                   │
│  │ Telegram │ │  Lark    │  ... (Slack, Discord TBD)        │
│  │  Plugin  │ │  Plugin  │                                   │
│  └────┬─────┘ └────┬─────┘                                   │
│       └────────────┴────────────┘                           │
│                    │                                         │
│  Responsibilities: Receive platform messages/callbacks → Convert to unified format → Send response        │
│  Not concerned with: Agent type, business logic                               │
└────────────────────┼────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 2: Gateway                        │
│                     (Business Logic Layer)                  │
├─────────────────────────────────────────────────────────────┤
│  ActionExecutor: System Action processing, conversation routing                  │
│  SessionManager: Session management, user authorization                          │
│  PairingService: Pairing code generation and validation                            │
│  ChannelMessageService: Message streaming processing                              │
│                                                              │
│  Responsibilities: System Action processing, conversation routing, session management, permission control       │
│  Not concerned with: Platform details, Agent implementation details                           │
└────────────────────┼────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 3: Agent                           │
│                     (AI Processing Layer)                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │ Gemini  │  │   ACP   │  │  Codex  │                      │
│  │  Agent  │  │  Agent  │  │  Agent  │                      │
│  └─────────┘  └─────────┘  └─────────┘                      │
│                                                              │
│  Responsibilities: Communicate with AI services, manage conversation context, return unified response         │
│  Not concerned with: Message source platform, system-level operations                           │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
Inbound Flow:
  Platform Message → Plugin(transform) → ActionExecutor(route) → Agent(process)

  Detailed Flow:
  1. Plugin receives platform message → toUnifiedIncomingMessage()
  2. PluginManager calls messageHandler → ActionExecutor.handleMessage()
  3. ActionExecutor routes by Action type:
     - Platform Action → Plugin handles internally
     - System Action → SystemActions handles
     - Chat Action → ChannelMessageService → Agent

Outbound Flow:
  Agent Response → ChannelEventBus → ChannelMessageService → ActionExecutor → Plugin(transform) → Platform send

  Detailed Flow:
  1. Agent Worker sends message → ChannelEventBus.emitAgentMessage()
  2. ChannelMessageService listens to event → handleAgentMessage()
  3. transformMessage + composeMessage → StreamCallback
  4. ActionExecutor calls context.sendMessage/editMessage()
  5. Plugin transforms message format → sendMessage/editMessage()
```

---

## 3. Plugin System Design

### 3.1 Plugin Responsibility Boundaries

| Plugin Responsible For                         | Plugin Not Responsible For                 |
| ---------------------------------------------- | ------------------------------------------ |
| Connect to platform API                        | Agent scheduling and execution             |
| Receive messages → Convert to unified format   | Session management and persistence         |
| Unified format → Convert to platform messages  | User authentication and permission control |
| Handle platform-specific commands              | Message routing decisions                  |
| Streaming message updates (edit sent messages) |                                            |

### 3.2 Plugin Lifecycle

```
created → initializing → ready → starting → running → stopping → stopped
                ↓                    ↓           ↓
              error ←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
```

| State          | Description                               |
| -------------- | ----------------------------------------- |
| `created`      | Plugin instance created                   |
| `initializing` | Validating config and initializing        |
| `ready`        | Initialization complete, waiting to start |
| `starting`     | Connecting to platform                    |
| `running`      | Running normally                          |
| `stopping`     | Disconnecting                             |
| `stopped`      | Stopped                                   |
| `error`        | Error occurred                            |

### 3.3 Plugin Interface (BasePlugin Abstract Class)

| Interface Method       | Direction               | Description                            |
| ---------------------- | ----------------------- | -------------------------------------- |
| `initialize(config)`   | PluginManager → Plugin  | Initialize plugin configuration        |
| `start()`              | PluginManager → Plugin  | Start platform connection              |
| `stop()`               | PluginManager → Plugin  | Stop platform connection               |
| `sendMessage(...)`     | ActionExecutor → Plugin | Send message to platform               |
| `editMessage(...)`     | ActionExecutor → Plugin | Edit sent message (streaming update)   |
| `getStatus()`          | PluginManager → Plugin  | Get plugin status                      |
| `getActiveUserCount()` | PluginManager → Plugin  | Get active user count                  |
| `getBotInfo()`         | PluginManager → Plugin  | Get Bot info                           |
| `onInitialize()`       | Subclass implementation | Platform-specific initialization logic |
| `onStart()`            | Subclass implementation | Platform-specific start logic          |
| `onStop()`             | Subclass implementation | Platform-specific stop logic           |

### 3.4 Unified Message Format

**Inbound Message (Platform → System)** - `IUnifiedIncomingMessage`

| Field              | Description                                   |
| ------------------ | --------------------------------------------- |
| `id`               | System-generated unique ID                    |
| `platform`         | Source platform (telegram/lark/slack/discord) |
| `chatId`           | Chat ID                                       |
| `user`             | User info (id, username, displayName)         |
| `content`          | Message content (type, text, attachments)     |
| `timestamp`        | Timestamp                                     |
| `replyToMessageId` | Replied message ID (optional)                 |
| `action`           | Action info (when button callback)            |
| `raw`              | Platform raw message (optional)               |

**Outbound Message (System → Platform)** - `IUnifiedOutgoingMessage`

| Field              | Description                                          |
| ------------------ | ---------------------------------------------------- |
| `type`             | Message type (text/image/file/buttons)               |
| `text`             | Text content                                         |
| `parseMode`        | Parse mode (HTML/Markdown/MarkdownV2)                |
| `buttons`          | Inline button group (optional)                       |
| `keyboard`         | Reply Keyboard (optional)                            |
| `replyMarkup`      | Platform-specific Markup (optional, e.g., Lark Card) |
| `replyToMessageId` | Replied message ID (optional)                        |
| `imageUrl`         | Image URL (image type)                               |
| `fileUrl`          | File URL (file type)                                 |
| `fileName`         | File name (file type)                                |
| `silent`           | Silent send (optional)                               |

### 3.5 Steps to Extend New Platform

1. Create `src/channels/plugins/[platform]/` directory
2. Implement `[Platform]Plugin` extending `BasePlugin`
3. Implement `[Platform]Adapter` for message conversion (toUnifiedIncomingMessage, to[Platform]SendParams)
4. Register plugin in `ChannelManager` constructor: `registerPlugin('platform', PlatformPlugin)`
5. Add platform type to `PluginType` in `types.ts`
6. Add settings page UI
7. Add i18n translations
8. Implement platform-specific interaction components (e.g., Keyboard, Card)

---

## 4. Implemented Platforms

### 4.1 Telegram Integration

#### Technology Selection

| Item         | Choice                 | Description                      |
| ------------ | ---------------------- | -------------------------------- |
| Bot Library  | grammY                 | Used by Clawdbot, elegant API    |
| Runtime Mode | Polling (long polling) | Automatic reconnection mechanism |

### 4.1 Technology Selection

| Item         | Choice                         | Description                   |
| ------------ | ------------------------------ | ----------------------------- |
| Bot Library  | grammY                         | Used by Clawdbot, elegant API |
| Runtime Mode | Polling (dev) / Webhook (prod) | Configurable                  |

#### Bot Configuration Process

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Create Bot                                            │
│   User in Telegram → @BotFather → /newbot → Get Token      │
├─────────────────────────────────────────────────────────────┤
│ Step 2: Configure Token                                          │
│   AionUi Settings page → Paste Token → Verify → Save               │
├─────────────────────────────────────────────────────────────┤
│ Step 3: Start Bot                                            │
│   Toggle switch → Bot starts listening                                   │
├─────────────────────────────────────────────────────────────┤
│ Step 4: User Pairing (see security mechanism below)                          │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration Items

| Config Item    | Type              | Description                                    |
| -------------- | ----------------- | ---------------------------------------------- |
| Bot Token      | string            | Get from @BotFather                            |
| Runtime Mode   | polling / webhook | Polling suitable for dev                       |
| Webhook URL    | string            | Required only for webhook mode                 |
| Pairing Mode   | boolean           | Whether pairing code authorization is required |
| Rate Limit     | number            | Max messages per minute                        |
| Group @Mention | boolean           | Whether @bot is required in groups to respond  |
| Default Agent  | gemini            | Fixed Gemini for MVP phase                     |

#### Pairing Security Mechanism (Clawdbot Pattern)

**Core Principle**: Approval operations are completed on the user's local device, not in Telegram

```
┌─────────────────────────────────────────────────────────────┐
│ ① User initiates in Telegram                                     │
│    User → @YourBot: /start or any message                      │
├─────────────────────────────────────────────────────────────┤
│ ② Bot returns pairing request                                         │
│    Bot → User:                                             │
│    "👋 Welcome to Aion Assistant!                                │
│     Your123                                     │
│     Please approve this pairing in Aion pairing code: ABCUi:                              │
│     Settings → Telegram → Pending Requests → [Approve]"                │
├─────────────────────────────────────────────────────────────┤
│ ③ AionUi shows pending approval request                                    │
│    Settings page displays: Username, Pairing Code, Request Time, [Approve]/[Reject]   │
├─────────────────────────────────────────────────────────────┤
│ ④ User clicks [Approve] in AionUi                                │
├─────────────────────────────────────────────────────────────┤
│ ⑤ Bot notifies pairing success                                         │
│    Bot → User: "✅ Pairing successful! You can start chatting now"           │
└─────────────────────────────────────────────────────────────┘
```

**Security Measures**

| Mechanism               | Description                                 |
| ----------------------- | ------------------------------------------- |
| Pairing Code            | 6-digit random code, 10 min validity        |
| Local Approval          | Must be approved in AionUi, not in Telegram |
| User Whitelist          | Only authorized users can use               |
| Rate Limit              | Prevent abuse                               |
| Token Encrypted Storage | Use bcrypt encryption                       |

#### Message Conversion Rules

**Inbound Conversion (Telegram → Unified)**

| Telegram Message Type | Unified Message content.type        |
| --------------------- | ----------------------------------- |
| `message:text`        | `text` or `command` (starts with /) |
| `message:photo`       | `image`                             |
| `message:document`    | `file`                              |
| `message:voice`       | `audio`                             |

**Outbound Conversion (Unified → Telegram)**

| Unified Message type | Telegram API                      |
| -------------------- | --------------------------------- |
| `text`               | `sendMessage`                     |
| `image`              | `sendPhoto`                       |
| `file`               | `sendDocument`                    |
| `buttons`            | `sendMessage` + `inline_keyboard` |

**Special Handling**

| Scenario           | Handling                                              |
| ------------------ | ----------------------------------------------------- |
| Streaming Response | Use `editMessageText` to update message, add ▌ cursor |
| Markdown           | Escape special characters, use `parse_mode: Markdown` |
| @Mention Removal   | Clean up `@bot_username` from messages                |
| Group Filtering    | Check if @mention is included (configurable)          |

### 4.2 Lark/Feishu Integration

#### Technology Selection

| Item         | Choice                                      | Description           |
| ------------ | ------------------------------------------- | --------------------- |
| SDK          | @larksuiteoapi/node-sdk                     | Official SDK          |
| Runtime Mode | WebSocket long connection                   | No public URL needed  |
| Domain       | Feishu (configurable to Lark international) | Default Feishu domain |

#### Bot Configuration Process

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Create App                                            │
│   Create enterprise self-built app on Feishu Open Platform → Get App ID and App Secret │
├─────────────────────────────────────────────────────────────┤
│ Step 2: Configure Permissions                                            │
│   App Permissions → Enable "Send and receive single chat/group messages"          │
├─────────────────────────────────────────────────────────────┤
│ Step 3: Configure Event Subscription                                        │
│   Event Subscription → Subscribe to "Receive message" event → Configure encryption key (optional)      │
├─────────────────────────────────────────────────────────────┤
│ Step 4: Configure Credentials                                            │
│   AionUi Settings page → Paste App ID, App Secret → Verify → Save   │
├─────────────────────────────────────────────────────────────┤
│ Step 5: Start Bot                                            │
│   Toggle switch → Bot connects via WebSocket and starts listening               │
├─────────────────────────────────────────────────────────────┤
│ Step 6: User Pairing (see security mechanism below)                          │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration Items

| Config Item        | Type    | Description                                    |
| ------------------ | ------- | ---------------------------------------------- |
| App ID             | string  | Get from Feishu Open Platform                  |
| App Secret         | string  | Get from Feishu Open Platform                  |
| Encrypt Key        | string  | Event encryption key (optional)                |
| Verification Token | string  | Event verification token (optional)            |
| Pairing Mode       | boolean | Whether pairing code authorization is required |
| Rate Limit         | number  | Max messages per minute                        |
| Default Agent      | gemini  | Fixed Gemini for MVP phase                     |

#### Pairing Security Mechanism

Same as Telegram, using local approval mode. Pairing code is sent to user via Lark message, user approves in AionUi.

#### Message Conversion Rules

**Inbound Conversion (Lark → Unified)**

| Lark Message Type | Unified Message content.type        |
| ----------------- | ----------------------------------- |
| `message:text`    | `text` or `command` (starts with /) |
| `message:image`   | `photo`                             |
| `message:file`    | `document`                          |
| `message:audio`   | `audio`                             |
| Card Action       | `action` (via extractCardAction)    |

**Outbound Conversion (Unified → Lark)**

| Unified Message type | Lark API                   |
| -------------------- | -------------------------- |
| `text`               | `im.message.create`        |
| `buttons`            | `im.message.create` + Card |
| Interactive Card     | Use Lark Card format       |

**Special Handling**

| Scenario            | Handling                                                   |
| ------------------- | ---------------------------------------------------------- |
| Streaming Response  | Use `im.message.update` to update message                  |
| HTML to Markdown    | convertHtmlToLarkMarkdown() convert HTML to Lark Markdown  |
| Card Interaction    | Use Lark Card format, support buttons, confirmations, etc. |
| Event Deduplication | 5-minute event cache, prevent duplicate processing         |

---

## 5. Interaction Design

### 5.1 Design Principles

**Buttons first, commands preserved**: Regular users operate via buttons, advanced users can use commands

### 5.2 Telegram Interaction Components

| Type                | Description                   | Applicable Scenarios                     |
| ------------------- | ----------------------------- | ---------------------------------------- |
| **Inline Keyboard** | Buttons below message         | Operation confirmation, option selection |
| **Reply Keyboard**  | Replace input method keyboard | Common operation shortcuts               |
| **Menu Button**     | Left of chat input box        | Fixed function entry                     |

### 5.3 Interaction Scenario Design

**Scenario 1: First Use/Pairing**

```
Bot Message:
┌─────────────────────────────────────────┐
│ 👋 Welcome to Aion Assistant!                 │
│                                          │
│ 🔗 Pairing Code: ABC123                       │
│ Please approve this pairing in AionUi settings            │
│                                          │
│ [📖 User Guide]  [❓ Get Help]            │
└─────────────────────────────────────────┘
```

**Scenario 2: After Successful Pairing (Reply Keyboard Permanent)**

```
┌─────────────────────────────────────────┐
│ ... Conversation content ...                        │
├─────────────────────────────────────────┤
│ Reply Keyboard (Permanent shortcuts)           │
│ [🆕 New Chat] [📊 Status] [❓ Help]         │
├─────────────────────────────────────────┤
│ [Enter message...]                   [Send]  │
└─────────────────────────────────────────┘
```

**Scenario 3: AI Response with Action Buttons**

````
Bot Message:
┌─────────────────────────────────────────┐
│ Here's a quicksort implementation:               │
│                                          │
│ ```python                                │
│ def quicksort(arr):                      │
│     ...                                  │
│ ```                                      │
│                                          │
│ [📋 Copy] [🔄 Regenerate] [💬 Continue]       │
└─────────────────────────────────────────┘
````

**Scenario 4: Settings Page (Card-style Selection)**

```
Bot Message:
┌─────────────────────────────────────────┐
│ ⚙️ Settings                                 │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 🤖 AI Model                          │ │
│ │ Current: Gemini 1.5 Pro                │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │ 💬 Conversation Style                         │ │
│ │ Current: Professional                          │ │
│ └─────────────────────────────────────┘ │
│                                          │
│ [← Back]                                │
└─────────────────────────────────────────┘
```

### 5.4 Button and Command Reference

| Command (hidden, preserved) | Button (visible to user) |
| --------------------------- | ------------------------ |
| `/start`                    | Auto-trigger             |
| `/new`                      | 🆕 New Chat              |
| `/status`                   | 📊 Status                |
| `/help`                     | ❓ Help                  |

---

## 6. Unified Action Processing Mechanism

### 6.1 Design Goal

Commands and button callbacks use unified processing to avoid duplicate logic and facilitate multi-platform extension

### 6.2 Action Classification

| Type                | Description                                        | Handler                  |
| ------------------- | -------------------------------------------------- | ------------------------ |
| **Platform Action** | Platform-specific operations (auth, pairing, etc.) | Plugin internal handling |
| **System Action**   | Platform-independent system-level operations       | Gateway ActionHandler    |
| **Chat Action**     | Messages requiring Agent processing                | AgentRouter → Agent      |

```
User Input
    │
    ├─→ Platform Action → Plugin handles internally (not entering Gateway)
    │       Example: Telegram pairing, Slack OAuth, Discord invite
    │
    ├─→ System Action → Gateway ActionHandler → Unified handling
    │       Example: Session management, settings, help
    │
    └─→ Chat Action → AgentRouter → Gemini/ACP/Codex
```

### 6.3 System Action List (Platform-independent)

| Category                | Action                  | Description                |
| ----------------------- | ----------------------- | -------------------------- |
| **Session Management**  | `session.new`           | Create new session         |
|                         | `session.status`        | View current status        |
|                         | `session.list`          | Session list (extension)   |
|                         | `session.switch`        | Switch session (extension) |
| **Settings Operations** | `settings.show`         | Show settings menu         |
|                         | `settings.model.list`   | Show model list            |
|                         | `settings.model.select` | Select model               |
|                         | `settings.agent.select` | Switch Agent (extension)   |
| **Help Info**           | `help.show`             | Show help                  |
| **Navigation**          | `nav.back`              | Go back                    |
|                         | `nav.cancel`            | Cancel current operation   |

### 6.4 Platform Action Examples (Each Plugin implements internally)

| Platform     | Action            | Description                  |
| ------------ | ----------------- | ---------------------------- |
| **Telegram** | `pairing.show`    | Show pairing code            |
|              | `pairing.refresh` | Refresh pairing code         |
| **Slack**    | `oauth.start`     | Initiate OAuth authorization |
|              | `oauth.callback`  | OAuth callback handling      |
| **Discord**  | `invite.generate` | Generate invite link         |

> **Note**: Platform Actions are handled internally by each Plugin, not through Gateway ActionHandler

### 6.5 Chat Action List

| Category               | Action            | Description            | Routes To             |
| ---------------------- | ----------------- | ---------------------- | --------------------- |
| **Send Message**       | `chat.send`       | User sends new message | Current session Agent |
| **Message Operations** | `chat.regenerate` | Regenerate answer      | Current session Agent |
|                        | `chat.continue`   | Continue generation    | Current session Agent |
|                        | `chat.stop`       | Stop generation        | Current session Agent |

### 6.6 Action Data Structure

```
UnifiedAction {
  action: string          // Action type
  params?: object         // Optional parameters
  context: {
    platform: string      // Source platform
    userId: string        // User ID
    chatId: string        // Chat ID
    messageId?: string    // Trigger message ID
    sessionId?: string    // Current session ID
  }
}
```

### 6.7 Button Callback Data Format

```
Format: action:param1=value1,param2=value2

Examples:
• "session.new"
• "settings.model.select:id=gemini-pro"
• "chat.regenerate:msg=abc123"
```

### 6.8 Unified Response Format

```
ActionResponse {
  text?: string                    // Text content
  parseMode?: 'plain' | 'markdown' // Parse mode
  buttons?: ActionButton[][]       // Inline buttons
  keyboard?: ActionButton[][]      // Reply Keyboard
  behavior: 'send' | 'edit' | 'answer'  // Response behavior
  toast?: string                   // Toast notification
}
```

---

## 7. Session Management

### 7.1 Session and Agent Relationship

```
Session {
  id: string              // Session ID
  platform: string        // Source platform
  userId: string          // User ID
  chatId: string          // Chat ID

  // Agent configuration
  agentType: string       // gemini / acp / codex
  agentConfig: {
    modelId?: string      // Model ID
  }

  // Session state
  status: string          // active / idle / error
  context: object         // Agent session context

  // Metadata
  createdAt: number
  lastActiveAt: number
}
```

### 7.2 MVP Phase Session Strategy

| Item            | MVP Implementation                   |
| --------------- | ------------------------------------ |
| Session Mode    | Single active session                |
| New Session     | Click 🆕 button clears context       |
| Session Storage | Independent from AionUi GUI sessions |
| Agent           | Fixed Gemini                         |
| Model           | Use AionUi default configuration     |

### 7.3 Future Extensions

| Item            | Extension Content                                |
| --------------- | ------------------------------------------------ |
| Multi-session   | Support `session.list` / `session.switch`        |
| Agent Switching | Support `settings.agent.select`                  |
| Model Switching | Support dynamic model selection                  |
| Session Sync    | Associate Telegram sessions with AionUi sessions |

---

## 8. Message Streaming Processing Architecture

### 8.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Worker (Gemini/ACP/Codex)              │
│                    (Agent Worker process)                        │
├─────────────────────────────────────────────────────────────────┤
│  Send message events to IPC Bridge                                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ChannelEventBus                               │
│                    (Global event bus - singleton)               │
├─────────────────────────────────────────────────────────────────┤
│  emitAgentMessage(conversationId, data)                          │
│  onAgentMessage(handler) → () => void (cleanup)                  │
│                                                                  │
│  Event type: 'channel.agent.message'                               │
│  Data structure: IAgentMessageEvent { ...IResponseMessage, conv_id }   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ChannelMessageService                         │
│                    (Message service - singleton)                │
├─────────────────────────────────────────────────────────────────┤
│  initialize() {                                                  │
│    // Register global event listener when service initializes                               │
│    channelEventBus.onAgentMessage(this.handleAgentMessage);    │
│  }                                                               │
│                                                                  │
│  handleAgentMessage(event) {                                     │
│    // Handle special events: start, finish, error                         │
│    // Use transformMessage + composeMessage to merge messages            │
│    // Callback: callback(TMessage, isInsert)                     │
│  }                                                               │
│                                                                  │
│  sendMessage(sessionId, conversationId, text, callback) {        │
│    // Only send message, don't handle listening                                     │
│    // Call Agent Task via WorkerManage                          │
│  }                                                               │
│                                                                  │
│  Internal state:                                                       │
│    activeStreams: Map<conversationId, IStreamState>              │
│    messageListMap: Map<conversationId, TMessage[]>               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ActionExecutor                                │
│                    (Business executor)                          │
├─────────────────────────────────────────────────────────────────┤
│  handleChatMessage(context, text) {                              │
│    messageService.sendMessage(                                   │
│      sessionId, conversationId, text,                             │
│      (message: TMessage, isInsert: boolean) => {                 │
│        const outgoing = convertTMessageToOutgoing(message, platform); │
│        if (isInsert) context.sendMessage(outgoing);              │
│        else context.editMessage(msgId, outgoing);                │
│      }                                                           │
│    );                                                            │
│  }                                                               │
│                                                                  │
│  convertTMessageToOutgoing(message, platform) {                  │
│    // TMessage → IUnifiedOutgoingMessage                         │
│    // Format text by platform (HTML/Markdown)                        │
│    // text → display content                                            │
│    // tips → tips with icons                                          │
│    // tool_group → tool status list                                  │
│  }                                                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Plugin (Telegram/Lark)                        │
│                    (Platform plugin)                            │
├─────────────────────────────────────────────────────────────────┤
│  sendMessage(chatId, message: IUnifiedOutgoingMessage)           │
│  editMessage(chatId, messageId, message: IUnifiedOutgoingMessage)│
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Event Type Handling

| Event Type          | Source                  | Handling                                     |
| ------------------- | ----------------------- | -------------------------------------------- |
| `start`             | Agent starts responding | Reset message list                           |
| `content`           | Streaming text block    | transformMessage → composeMessage → callback |
| `tool_group`        | Tool call status        | Merge into existing tool_group or add new    |
| `finish`/`finished` | Response complete       | Resolve promise, clean up status             |
| `error`             | Error occurred          | Reject promise, clean up status              |
| `thought`           | Thinking process        | Ignore (transformMessage returns undefined)  |

### 8.3 Message Merge Strategy (composeMessage)

| Message Type | Merge Rule                                                 |
| ------------ | ---------------------------------------------------------- |
| `text`       | Same msg_id accumulates content, different msg_id adds new |
| `tool_group` | Merge tool status updates by callId                        |
| `tool_call`  | Merge by callId                                            |
| `tips`       | Add directly                                               |

### 8.4 Message Callback Parameters

```typescript
type StreamCallback = (chunk: TMessage, isInsert: boolean) => void;

// isInsert = true:  New message, call sendMessage to send new message
// isInsert = false: Update message, call editMessage to edit existing message
```

### 8.5 Throttle Control

| Parameter          | Value     | Description                          |
| ------------------ | --------- | ------------------------------------ |
| UPDATE_THROTTLE_MS | 500ms     | Minimum interval for message editing |
| Send new message   | Unlimited | Send immediately when isInsert=true  |
| Edit message       | Throttled | Apply throttle when isInsert=false   |

- [ ] Use existing Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Need new Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Component internal state only (useState/useReducer)
- [ ] Need persistent storage

### 8.6 Key Design Principles

1. **Separation of Event Listening and Message Sending**
   - Event listening completed during service initialization (`initialize()`)
   - `sendMessage()` only responsible for sending messages, not handling listening

2. **Global Event Bus Decoupling**
   - `ChannelMessageService` doesn't directly interact with Agent Task
   - Decoupled via `ChannelEventBus` global event bus

3. **Unified Message Format**
   - Internal use `TMessage` unified message format
   - Convert to `IUnifiedOutgoingMessage` when outputting

---

## 9. Agent Interface Specification

### 8.1 Capabilities Each Agent Must Implement

| Capability      | Description                   |
| --------------- | ----------------------------- |
| `sendMessage`   | Send message and get response |
| `streamMessage` | Stream send message           |
| `regenerate`    | Regenerate previous response  |
| `continue`      | Continue generation           |
| `stop`          | Stop current generation       |
| `getContext`    | Get session context           |
| `clearContext`  | Clear session context         |

### 8.2 Agent Response Format

```
AgentResponse {
  type: 'text' | 'stream_start' | 'stream_chunk' | 'stream_end' | 'error'
  text?: string
  chunk?: string
  error?: { code: string, message: string }
  metadata?: {
    model?: string
    tokensUsed?: number
    duration?: number
  }
  suggestedActions?: ActionButton[]
}
```

---

## 9. File Structure (Actual Implementation)

```
src/channels/
├── core/                          # Core modules
│   │   ├── ChannelManager.ts          # Unified manager (singleton)
│   │   └── SessionManager.ts          # Session management
│   │
│   ├── gateway/                       # Gateway layer
│   │   ├── PluginManager.ts           # Plugin lifecycle management
│   │   └── ActionExecutor.ts          # Action executor (routing, message processing)
│   │
│   ├── actions/                       # Action processing (platform-independent)
│   │   ├── types.ts                   # Action/Response type definitions
│   │   ├── SystemActions.ts          # System Actions (session, settings, help)
│   │   ├── ChatActions.ts            # Chat Actions (send, regenerate, etc.)
│   │   └── PlatformActions.ts        # Platform Actions (pairing, etc.)
│   │
│   ├── agent/                         # Agent integration
│   │   ├── ChannelEventBus.ts        # Global event bus
│   │   └── ChannelMessageService.ts  # Message streaming processing service
│   │
│   ├── pairing/                       # Pairing service
│   │   └── PairingService.ts         # Pairing code generation and validation (platform-independent)
│   │
│   ├── plugins/                       # Plugin directory
│   │   ├── BasePlugin.ts              # Plugin abstract base class
│   │   ├── telegram/
│   │   │   ├── TelegramPlugin.ts      # Telegram plugin
│   │   │   ├── TelegramAdapter.ts     # Message adapter
│   │   │   └── TelegramKeyboards.ts   # Keyboard components
│   │   └── lark/
│   │       ├── LarkPlugin.ts          # Lark plugin
│   │       ├── LarkAdapter.ts         # Message adapter
│   │       └── LarkCards.ts           # Card components
│   │
│   ├── utils/                         # Utility functions
│   │   └── credentialCrypto.ts        # Credential encryption
│   │
│   └── types.ts                       # Type definitions
```

---

## 10. Database Design

| Table Name                | Purpose                                  |
| ------------------------- | ---------------------------------------- |
| `assistant_plugins`       | Plugin configuration (Token, mode, etc.) |
| `assistant_users`         | Authorized user list                     |
| `assistant_sessions`      | User session associations                |
| `assistant_pairing_codes` | Pending pairing approval requests        |

---

## 11. External Dependencies

| Dependency Package        | Purpose           | Description                   |
| ------------------------- | ----------------- | ----------------------------- |
| `grammy`                  | Telegram Bot      | Used by Clawdbot, elegant API |
| `@larksuiteoapi/node-sdk` | Lark/Feishu Bot   | Official SDK                  |
| `@slack/bolt`             | Slack Bot (TBD)   | Official SDK                  |
| `discord.js`              | Discord Bot (TBD) | Official SDK                  |

---

## 12. Implementation Status

### 12.1 Implemented Features

#### Telegram

- [x] Bot Token configuration and verification
- [x] Bot start/stop control (Polling mode, automatic reconnection)
- [x] Pairing code generation and local approval process
- [x] Authorized user management
- [x] Button interaction (Reply Keyboard + Inline Keyboard)
- [x] Conversation with Gemini/ACP/Codex Agent
- [x] New session functionality
- [x] Streaming message response (editMessage update)
- [x] Tool confirmation interaction
- [x] Error recovery mechanism

#### Lark/Feishu

- [x] App ID/Secret configuration and verification
- [x] Bot start/stop control (WebSocket long connection)
- [x] Pairing code generation and local approval process
- [x] Authorized user management
- [x] Card interaction (buttons, confirmations, etc.)
- [x] Conversation with Gemini/ACP/Codex Agent
- [x] New session functionality
- [x] Streaming message response (updateMessage update)
- [x] Tool confirmation interaction (Card format)
- [x] Event deduplication mechanism (5 minute cache)
- [x] HTML to Lark Markdown conversion

#### Core Features

- [x] ChannelManager unified management
- [x] PluginManager plugin lifecycle management
- [x] SessionManager session management
- [x] PairingService pairing service
- [x] ActionExecutor Action routing and execution
- [x] ChannelMessageService message streaming processing
- [x] ChannelEventBus global event bus
- [x] Credential encrypted storage
- [x] Multi-platform unified message format

### 12.2 Security Acceptance

- [x] Pairing code expires in 10 minutes
- [x] Must be approved locally in AionUi
- [x] Unauthorized users cannot use
- [x] Token/credential encrypted storage
- [ ] Rate limiting (TBD)

### 12.3 Compatibility

- [x] Runs normally on macOS
- [x] Runs normally on Windows
- [x] Multi-language support (i18n)

---

## 13. Future Extension Roadmap

| Phase       | Content                                      | Status                 |
| ----------- | -------------------------------------------- | ---------------------- |
| **Phase 1** | Telegram + Lark integration                  | ✅ Completed           |
| **Phase 2** | Multi-session management, session switching  | 🔄 TBD                 |
| **Phase 3** | Agent switching (already supported, need UI) | 🔄 Partially Completed |
| **Phase 4** | Model dynamic switching                      | 🔄 TBD                 |
| **Phase 5** | Slack platform integration                   | 🔄 TBD                 |
| **Phase 6** | Discord platform integration                 | 🔄 TBD                 |
| **Phase 7** | Rate limiting                                | 🔄 TBD                 |
| **Phase 8** | Session sync with AionUi                     | 🔄 TBD                 |
| **Phase 9** | Headless independent service mode            | 🔄 TBD                 |

---

## Template Maintenance

- **Created**: 2025-01-27
- **Last Updated**: 2026-02-03
- **Applicable Version**: AionUi v1.7.8+
- **Maintainer**: Project Team

---

## Appendix: Key Implementation Details

### A.1 ChannelManager Initialization Process

```typescript
1. ChannelManager.getInstance().initialize()
   ├─ Initialize PluginManager
   ├─ Initialize SessionManager
   ├─ Initialize PairingService
   ├─ Initialize ActionExecutor
   └─ Initialize ChannelMessageService

2. Load plugin configurations from database
3. Call initialize() and start() for each enabled plugin
```

### A.2 Message Processing Process

```typescript
1. Plugin receives platform message
   └─ toUnifiedIncomingMessage() convert

2. PluginManager calls messageHandler
   └─ ActionExecutor.handleMessage()

3. ActionExecutor routes Action
   ├─ Platform Action → PlatformActions
   ├─ System Action → SystemActions
   └─ Chat Action → ChannelMessageService

4. ChannelMessageService.sendMessage()
   └─ Call Agent Task via WorkerManage

5. Agent response → ChannelEventBus
   └─ ChannelMessageService.handleAgentMessage()
   └─ ActionExecutor.handleAgentMessage()

6. ActionExecutor sends response
   └─ convertTMessageToOutgoing()
   └─ context.sendMessage() / context.editMessage()

7. Plugin converts and sends
   └─ toTelegramSendParams() / toLarkSendParams()
   └─ sendMessage() / editMessage()
```

### A.3 Pairing Code Generation

```typescript
// Use crypto.randomBytes for secure random code
const pairingCode = generatePairingCode(): string => {
  return crypto.randomBytes(3).toString('hex').toUpperCase(); // 6-character code
};

// Validity: 10 minutes
const PAIRING_CODE_EXPIRY_MS = 10 * 60 * 1000;
```

### A.4 Streaming Message Update Flow

```
Agent Token
    │
    ├─ "start" event → Create new message → sendMessage
    │
    ├─ "content" event → Update message → editMessage (throttled)
    │
    ├─ "tool_group" event → Update tool status → editMessage
    │
    └─ "finish" event → Complete → resolve promise
```

### A.5 Error Handling Strategy

| Error Type         | Handling                                      |
| ------------------ | --------------------------------------------- |
| Platform API Error | Retry with exponential backoff, max 3 retries |
| Agent Error        | Return error message to user, suggest retry   |
| Session Error      | Notify user, offer to create new session      |
| Network Error      | Auto-reconnect, queue pending messages        |

---

## Glossary

| Term          | Definition                                                       |
| ------------- | ---------------------------------------------------------------- |
| **Plugin**    | Platform adapter that handles specific IM platform communication |
| **Gateway**   | Business logic layer handling routing, session, actions          |
| **Channel**   | Abstraction layer for platform communication                     |
| **Pairing**   | Security mechanism linking IM user to AionUi account             |
| **Action**    | Unified command format for user interactions                     |
| **Streaming** | Real-time message delivery during Agent response generation      |

---

_Document Version: 1.7.8_
_Last Updated: 2026-02-03_
