# AionUi Feature Development Specification Template

> This template is used to standardize how feature development requirements are described to AI, ensuring the AI can accurately understand the task and follow project conventions.

---

## 1. Feature Overview

### 1.1 Basic Information

- **Feature Name**: [Concise Name]
- **Module**: [ ] Agent Layer [ ] Conversation System [ ] Preview System [ ] Settings System [ ] Workspace [ ] Other
- **Involved Processes**: [ ] Main Process [ ] Renderer [ ] WebServer [ ] Worker

### 1.2 Feature Description

[Describe the core purpose and value of the feature in 1-3 sentences]

### 1.3 User Scenarios

```
Trigger: [How the user triggers this feature]
Process: [How the system responds]
Result: [State after the feature is completed]
```

### 1.4 Data Flow

| Direction | Data Type | Description |
| --------- | --------- | ----------- |
| Input     |           |             |
| Output    |           |             |

---

## 2. Development Standards

### 2.1 Tech Stack Constraints

- **Framework**: Electron 37 + React 19 + TypeScript 5.8
- **UI Library**: Arco Design (@arco-design/web-react)
- **Icons**: Icon Park (@icon-park/react)
- **CSS**: UnoCSS atomic styling
- **State Management**: React Context (AuthContext / ConversationContext / ThemeContext / LayoutContext)
- **IPC Communication**: @office-ai/platform bridge system
- **Internationalization**: i18next + react-i18next
- **Database**: better-sqlite3

### 2.2 Naming Conventions

| Type              | Convention                | Example                                         |
| ----------------- | ------------------------- | ----------------------------------------------- |
| React Components  | PascalCase                | `MessageList.tsx`, `FilePreview.tsx`            |
| Hooks             | use prefix + PascalCase   | `useAutoScroll.ts`, `useColorScheme.ts`         |
| Bridge Files      | FeatureName + Bridge      | `conversationBridge.ts`, `databaseBridge.ts`    |
| Service Files     | FeatureName + Service     | `WebuiService.ts`                               |
| Interface Types   | I prefix                  | `ICreateConversationParams`, `IResponseMessage` |
| Type Aliases      | T prefix or direct naming | `TChatConversation`, `PresetAgentType`          |
| Constants         | UPPER_SNAKE_CASE          | `MAX_RETRY_COUNT`                               |
| Utility Functions | camelCase                 | `formatMessage`, `parseResponse`                |

### 2.3 File Location Standards

```
New files should be placed in the corresponding directory:

src/
├── agent/                        # AI agent implementations
│   ├── acp/                      # ACP protocol agent
│   ├── codex/                    # Codex agent
│   └── gemini/                   # Gemini agent
│
├── common/                       # Cross-process shared modules
│   ├── adapters/                 # API adapters
│   ├── types/                    # Shared type definitions
│   └── utils/                    # Shared utility functions
│
├── process/                      # Electron main process
│   ├── bridge/                   # IPC bridge definitions (24+)
│   ├── database/                 # SQLite database operations
│   ├── services/                 # Business logic services
│   └── task/                     # Task management
│
├── renderer/                     # React renderer process
│   ├── components/               # Reusable UI components
│   │   └── base/                 # Base components
│   ├── context/                  # React Context state
│   ├── hooks/                    # Custom Hooks (31+)
│   ├── pages/                    # Page components
│   │   ├── conversation/         # Conversation page
│   │   │   ├── preview/          # Preview panel
│   │   │   └── workspace/        # Workspace
│   │   ├── settings/             # Settings page (12+)
│   │   └── login/                # Login page
│   ├── messages/                 # Message rendering components
│   ├── i18n/locales/             # Internationalization text
│   ├── services/                 # Frontend services
│   └── utils/                    # Frontend utility functions
│
├── webserver/                    # Web server (WebUI mode)
│   ├── routes/                   # API routes
│   └── middleware/               # Middleware
│
├── worker/                       # Web Worker
│
└── types/                        # Global type definitions
```

### 2.4 Code Style (Prettier Config)

```json
{
  "semi": true,
  "singleQuote": true,
  "jsxSingleQuote": true,
  "trailingComma": "es5",
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### 2.5 Quality Requirements

- [ ] Complete TypeScript types, avoid using `any`
- [ ] Use bridge system for IPC communication
- [ ] Implement error boundary handling
- [ ] Support internationalization (use i18next `t()` function)
- [ ] Dark/Light theme compatible
- [ ] Responsive layout adaptation

### 2.6 Prohibited Items

- ❌ Direct use of `ipcMain` / `ipcRenderer`, must use bridge system
- ❌ Direct access to Node.js APIs in renderer process
- ❌ Hardcoded Chinese/English text, must use i18n keys
- ❌ Use inline styles, should use UnoCSS class names
- ❌ Direct DOM manipulation in components, use React ref
- ❌ Ignore TypeScript errors (`@ts-ignore`)

---

## 3. Implementation Architecture

### 3.1 Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User Interface (UI)                   │
│  React Components / Hooks / Context                     │
└─────────────────────┬───────────────────────────────────┘
                      │ IPC Bridge
┌─────────────────────▼───────────────────────────────────┐
│                   Main Process (Main)                    │
│  Bridge → Service → Database / External API             │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              Data Layer (Data)                           │
│  SQLite / LocalStorage / External Services              │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Files to Modify/Create

**Main Process (src/process/)**

| File Path | Operation            | Description |
| --------- | -------------------- | ----------- |
|           | [ ] New / [ ] Modify |             |

**Renderer Process (src/renderer/)**

| File Path | Operation            | Description |
| --------- | -------------------- | ----------- |
|           | [ ] New / [ ] Modify |             |

**Shared Modules (src/common/)**

| File Path | Operation            | Description |
| --------- | -------------------- | ----------- |
|           | [ ] New / [ ] Modify |             |

**Type Definitions (src/types/)**

| File Path | Operation            | Description |
| --------- | -------------------- | ----------- |
|           | [ ] New / [ ] Modify |             |

### 3.3 IPC Communication Design

To add new IPC channels, follow this pattern:

```typescript
// src/process/bridge/[Feature]Bridge.ts
import { bridge } from '@anthropic/platform';

export const [FeatureName] = {
  // Provider mode: Request-Response (similar to HTTP)
  [methodName]: bridge.buildProvider<TResponse, TParams>('[channel name]'),

  // Emitter mode: Event stream (for streaming data)
  [eventName]: bridge.buildEmitter<TData>('[channel name].stream'),
};

// Usage example:
// Renderer process call: const result = await [FeatureName].[methodName].request(params);
// Renderer process listener: [FeatureName].[eventName].on((data) => { ... });
```

### 3.4 State Management Design

- [ ] Use existing Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Need new Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Component internal state only (useState/useReducer)
- [ ] Need persistent storage

### 3.5 Internationalization Key Design

```json
// Add to src/renderer/i18n/locales/[lang].json
// Key naming convention: [module].[feature].[description]

{
  "conversation.export.title": "Export Conversation",
  "conversation.export.success": "Export Successful",
  "conversation.export.error": "Export Failed"
}
```

**Supported language files:**

- `zh-CN.json` - Simplified Chinese (required)
- `en-US.json` - English (required)
- `zh-TW.json` - Traditional Chinese
- `ja-JP.json` - Japanese
- `ko-KR.json` - Korean

---

## 4. Acceptance Criteria

### 4.1 Feature Acceptance

- [ ] [Specific feature point 1]
- [ ] [Specific feature point 2]
- [ ] [Specific feature point 3]

### 4.2 Edge Cases

- [ ] [Handling of exception scenario 1]
- [ ] [Handling of exception scenario 2]

### 4.3 Compatibility Acceptance

- [ ] Runs normally on macOS
- [ ] Runs normally on Windows
- [ ] Dark mode displays correctly
- [ ] Light mode displays correctly
- [ ] Language switching works correctly

### 4.4 Code Quality

- [ ] `npm run lint` has no errors
- [ ] `npm run build` builds successfully
- [ ] No TypeScript type errors
- [ ] No leftover console.log statements

---

## 5. Reference Materials

### 5.1 Similar Feature References

[List similar implementations available in the project]

| Feature | File Path | Description |
| ------- | --------- | ----------- |
|         |           |             |

### 5.2 Existing Module Dependencies

[List existing interfaces/components/hooks that need to be called]

| Module | Path | Purpose |
| ------ | ---- | ------- |
|        |      |         |

### 5.3 External Dependencies

[If new dependencies need to be introduced, list and explain]

| Package | Version | Purpose | Justification |
| ------- | ------- | ------- | ------------- |
|         |         |         |               |

### 5.4 Special Notes

[List special considerations during implementation]

---

## Usage Example

Below is a complete feature requirement example:

```markdown
## 1. Feature Overview

### 1.1 Basic Information

- **Feature Name**: Export Conversation as PDF
- **Module**: [x] Conversation System
- **Involved Processes**: [x] Main Process [x] Renderer Process

### 1.2 Feature Description

Allow users to export the current conversation as a PDF file, preserving message formatting, code highlighting, and images.

### 1.3 User Scenarios

Trigger: User clicks the "Export" button in the top-right corner of the conversation page, selects "Export as PDF"
Process: System collects conversation content, renders as HTML, converts to PDF
Result: Save dialog appears, user selects save location, PDF file is generated

### 3.2 Files to Modify/Create

**Main Process (src/process/)**
| File Path | Operation | Description |
|----------|------|------|
| src/process/bridge/exportBridge.ts | [x] New | PDF export IPC channel definition |
| src/process/services/ExportService.ts | [x] New | PDF generation logic |

**Renderer Process (src/renderer/)**
| File Path | Operation | Description |
|----------|------|------|
| src/renderer/pages/conversation/components/ChatHeader.tsx | [x] Modify | Add export dropdown menu |
| src/renderer/hooks/useExportPdf.ts | [x] New | Export functionality hook |

### 4.1 Feature Acceptance

- [ ] Export options menu appears when clicking export button
- [ ] Save dialog appears after selecting PDF
- [ ] Generated PDF contains complete conversation content
- [ ] Code blocks preserve syntax highlighting
- [ ] Images are correctly embedded in PDF

### 5.1 Similar Feature References

| Feature         | File Path                                             | Description                |
| --------------- | ----------------------------------------------------- | -------------------------- |
| Markdown Export | src/renderer/hooks/useExportMarkdown.ts               | Can reference export flow  |
| PDF Preview     | src/renderer/pages/conversation/preview/PdfViewer.tsx | Can reference PDF handling |
```

---

## Template Maintenance

- **Created**: 2025-01-27
- **Applicable Version**: AionUi v0.x+
- **Maintainer**: [Project Team]

Please sync modifications to this file and notify team members when updating the template.
