# Implementation Plan: Approach B — Native Rich Chat UI

## Goal

Add a **view mode toggle** to each Clutch session that switches between:
- **Terminal mode** (current) — raw xterm.js PTY rendering
- **Chat mode** (new) — rich React chat UI powered by Claude's stream-json protocol

The toggle should be per-session, allowing users to have some sessions in terminal mode
and others in chat mode simultaneously.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Clutch Desktop (Tauri v2)                                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  SessionContent.tsx                                        │  │
│  │                                                            │  │
│  │  ┌──────────────────────┐  ┌───────────────────────────┐  │  │
│  │  │  [Terminal] [Chat]   │  │  Session Settings          │  │  │
│  │  │  ← toggle per tab    │  │  (model, permissions, etc) │  │  │
│  │  └──────────────────────┘  └───────────────────────────┘  │  │
│  │                                                            │  │
│  │  viewMode === "terminal"        viewMode === "chat"        │  │
│  │  ┌──────────────────┐          ┌──────────────────────┐   │  │
│  │  │  Terminal.tsx     │          │  ChatView.tsx         │   │  │
│  │  │  (existing)       │          │  (NEW)                │   │  │
│  │  │                   │          │                       │   │  │
│  │  │  usePty hook      │          │  useClaude hook       │   │  │
│  │  │  xterm.js         │          │  (NEW)                │   │  │
│  │  │  PTY backend      │          │                       │   │  │
│  │  │                   │          │  ┌─────────────────┐  │   │  │
│  │  │                   │          │  │ MessageList     │  │   │  │
│  │  │                   │          │  │ ToolCallBlock   │  │   │  │
│  │  │                   │          │  │ DiffViewer      │  │   │  │
│  │  │                   │          │  │ PermissionDialog│  │   │  │
│  │  │                   │          │  │ InputBar        │  │   │  │
│  │  │                   │          │  └─────────────────┘  │   │  │
│  │  └──────────────────┘          └──────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Rust Backend (Tauri)                                      │  │
│  │                                                            │  │
│  │  PTY Manager (existing)     Claude Process Manager (NEW)   │  │
│  │  - spawn shell + claude     - spawn claude binary directly │  │
│  │  - raw byte I/O             - --output-format stream-json  │  │
│  │  - terminal emulation       - --input-format stream-json   │  │
│  │                              - NDJSON I/O                  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Phased Implementation

### Phase 1: Foundation (Week 1-2)

**Goal**: Bidirectional stream-json communication working end-to-end.

#### 1.1 Rust: Claude Process Manager

Create `src-tauri/src/claude_process.rs`:

```rust
// New module alongside pty.rs
// Spawns `claude` binary with stream-json flags
// Reads NDJSON from stdout, emits Tauri events
// Accepts JSON input via Tauri commands, writes to stdin

pub struct ClaudeProcessManager {
    child: Child,
    stdin_writer: Arc<Mutex<ChildStdin>>,
}

impl ClaudeProcessManager {
    pub fn spawn(
        working_dir: &str,
        model: Option<&str>,
        permission_mode: &str,
        initial_prompt: Option<&str>,
    ) -> Result<Self, String> {
        // Spawn: claude -p --output-format stream-json 
        //               --input-format stream-json
        //               --include-partial-messages
        //               --permission-mode <mode>
        //               [--model <model>]
    }
    
    pub fn start_reader(&self, app: AppHandle, session_id: String) {
        // Background thread: read stdout line by line
        // Parse each line as JSON
        // Emit "claude-message" Tauri event with parsed data
    }
    
    pub fn send_message(&self, json: &str) -> Result<(), String> {
        // Write JSON + newline to stdin
    }
    
    pub fn abort(&self) -> Result<(), String> {
        // Send abort message or kill process
    }
}
```

#### 1.2 Rust: Tauri Commands

Add to `src-tauri/src/commands.rs`:

```rust
#[tauri::command]
async fn claude_spawn(
    session_id: String,
    working_dir: String,
    model: Option<String>,
    permission_mode: String,
    initial_prompt: Option<String>,
    app: AppHandle,
) -> Result<(), String> { ... }

#[tauri::command]
async fn claude_send(
    session_id: String,
    message: String,  // JSON string
) -> Result<(), String> { ... }

#[tauri::command]
async fn claude_abort(
    session_id: String,
) -> Result<(), String> { ... }
```

#### 1.3 Frontend: useClaude Hook

Create `src/hooks/useClaude.ts`:

```typescript
interface UseClaudeOptions {
  sessionId: string;
  workingDir: string;
  onMessage: (msg: ClaudeMessage) => void;
  onError: (error: string) => void;
  onExit: () => void;
}

interface UseClaude {
  spawn: (prompt?: string) => Promise<void>;
  sendMessage: (content: string) => Promise<void>;
  respondPermission: (requestId: string, allowed: boolean) => Promise<void>;
  abort: () => Promise<void>;
}

export function useClaude(options: UseClaudeOptions): UseClaude {
  // Listen for "claude-message" Tauri events
  // Parse message types and dispatch to onMessage callback
  // Provide send/respond/abort functions that invoke Tauri commands
}
```

#### 1.4 Frontend: Message Types

Create `src/lib/claude-types.ts`:

```typescript
// Union type of all possible messages from the stream-json protocol
type ClaudeMessage =
  | InitMessage
  | AssistantTextMessage
  | AssistantToolUseMessage
  | ToolResultMessage
  | PermissionRequestMessage
  | ContentBlockDelta
  | ResultMessage
  | ErrorMessage;

interface InitMessage {
  type: "system";
  subtype: "init";
  session_id: string;
  tools: string[];
  model: string;
  cwd: string;
}

interface AssistantTextMessage {
  type: "assistant";
  message: {
    id: string;
    content: Array<{ type: "text"; text: string }>;
    model: string;
    stop_reason: string;
    usage: { input_tokens: number; output_tokens: number };
  };
}

// ... etc for all message types
```

### Phase 2: Chat UI Components (Week 2-3)

**Goal**: Core chat interface rendering messages from stream-json.

#### 2.1 ChatView.tsx — Main Container

```
┌─────────────────────────────────────┐
│  Model: claude-sonnet-4-6  │ ⚙️    │  ← Header bar
├─────────────────────────────────────┤
│                                     │
│  👤 User                            │  ← MessageList
│  "Explain the auth module"          │
│                                     │
│  🤖 Claude                          │
│  "The auth module handles..."       │
│                                     │
│  🔧 Tool: Read                      │  ← ToolCallBlock
│  📄 src/auth.ts (lines 1-50)       │
│  [▼ Show output]                    │
│                                     │
│  🤖 Claude                          │
│  "Based on reading the file..."     │
│                                     │
│  ⚠️ Permission: Edit               │  ← PermissionDialog
│  📄 src/auth.ts                     │
│  [View Diff] [Accept] [Reject]      │
│                                     │
├─────────────────────────────────────┤
│  Type a message...          [Send]  │  ← InputBar
│  Mode: [Default ▼]  Context: 45%   │
└─────────────────────────────────────┘
```

#### 2.2 Component Breakdown

New components to create in `src/components/chat/`:

| Component | Purpose | Dependencies |
|-----------|---------|-------------|
| `ChatView.tsx` | Main container, manages message state | useClaude hook |
| `MessageList.tsx` | Scrollable message list | All message components |
| `UserMessage.tsx` | Renders user prompts | Markdown renderer |
| `AssistantMessage.tsx` | Renders Claude's text responses | Markdown renderer, syntax highlighting |
| `ToolCallBlock.tsx` | Collapsible tool invocation display | Tool-specific icons |
| `ToolResultBlock.tsx` | Tool output display | Code blocks, file content |
| `PermissionDialog.tsx` | Accept/reject permission requests | DiffViewer (for Edit tool) |
| `DiffViewer.tsx` | Side-by-side or inline diff | react-diff-viewer or custom |
| `InputBar.tsx` | Text input with send button | Permission mode selector |
| `StreamingText.tsx` | Incremental text rendering | For partial message display |
| `ChatHeader.tsx` | Model info, context usage, settings | Status indicators |

#### 2.3 Recommended Libraries

| Library | Purpose | Alternative |
|---------|---------|-------------|
| `react-markdown` | Render markdown in messages | `marked` + `DOMPurify` |
| `react-syntax-highlighter` | Code blocks in messages | `shiki` or `prism-react-renderer` |
| `react-diff-viewer-continued` | Diff viewing for Edit tool | Custom diff with `diff` npm package |
| `cmdk` | Command palette for slash commands | Custom dropdown |

### Phase 3: View Mode Toggle (Week 3)

**Goal**: Seamless switching between terminal and chat modes.

#### 3.1 Session Model Update

Extend the Session type in `src/lib/sessions.ts`:

```typescript
interface Session {
  id: string;
  workingDir: string;
  command: string;
  // ... existing fields
  viewMode: "terminal" | "chat";  // NEW
}
```

#### 3.2 SessionContent.tsx Changes

```tsx
// In SessionContent.tsx, render based on viewMode:
{session.viewMode === "terminal" ? (
  <Terminal
    sessionId={session.id}
    workingDir={session.workingDir}
    command={session.command}
    isActive={isActive}
    onStatusChange={...}
  />
) : (
  <ChatView
    sessionId={session.id}
    workingDir={session.workingDir}
    isActive={isActive}
    onStatusChange={...}
  />
)}
```

#### 3.3 Toggle Button

Add a toggle to `TitleBar.tsx` or as a tab-level control:

```tsx
<ToggleGroup value={viewMode} onValueChange={setViewMode}>
  <ToggleGroupItem value="terminal">Terminal</ToggleGroupItem>
  <ToggleGroupItem value="chat">Chat</ToggleGroupItem>
</ToggleGroup>
```

### Phase 4: Polish & Advanced Features (Week 4+)

#### 4.1 Permission Flow
- Full permission request/response cycle
- Accept/reject with visual feedback
- "Always allow" option per tool

#### 4.2 Plan Mode
- Detect plan mode messages
- Render plan as editable markdown
- Allow inline comments before approval

#### 4.3 Slash Commands
- Command palette triggered by `/` in input
- Support: `/model`, `/compact`, `/usage`, `/clear`

#### 4.4 @-Mentions
- File picker autocomplete triggered by `@`
- Tauri command to list files in working directory
- Insert file reference into message

#### 4.5 Context Indicator
- Parse usage data from messages
- Show context window percentage
- Auto-compact notification

#### 4.6 Conversation History
- Persist chat messages alongside worktree data
- Resume conversations from session list

## New Dependencies

### Frontend (package.json)
```json
{
  "react-markdown": "^9.x",
  "react-syntax-highlighter": "^15.x",
  "react-diff-viewer-continued": "^4.x",
  "remark-gfm": "^4.x"
}
```

### Backend (Cargo.toml)
No new Rust dependencies needed — `std::process::Command`, `serde_json`, and
existing `tokio` are sufficient for spawning and managing the Claude process.

## File Organization

```
src/
├── components/
│   ├── chat/                    # NEW — all chat mode components
│   │   ├── ChatView.tsx         # Main chat container
│   │   ├── ChatHeader.tsx       # Model info, context usage
│   │   ├── MessageList.tsx      # Scrollable message list
│   │   ├── UserMessage.tsx      # User message bubble
│   │   ├── AssistantMessage.tsx  # Claude response bubble
│   │   ├── ToolCallBlock.tsx    # Tool invocation display
│   │   ├── ToolResultBlock.tsx  # Tool output display
│   │   ├── PermissionDialog.tsx # Permission request UI
│   │   ├── DiffViewer.tsx       # Code diff display
│   │   ├── InputBar.tsx         # Message input with controls
│   │   └── StreamingText.tsx    # Incremental text rendering
│   ├── Terminal.tsx             # Existing — unchanged
│   ├── SessionContent.tsx       # Modified — adds view mode logic
│   └── ...
├── hooks/
│   ├── useClaude.ts             # NEW — Claude stream-json bridge
│   ├── usePty.ts                # Existing — unchanged
│   └── ...
├── lib/
│   ├── claude-types.ts          # NEW — TypeScript types for protocol
│   ├── claude-parser.ts         # NEW — NDJSON parser utility
│   └── ...
src-tauri/src/
├── claude_process.rs            # NEW — Claude process manager
├── pty.rs                       # Existing — unchanged
├── commands.rs                  # Modified — add claude_* commands
└── ...
```

## Testing Strategy

1. **Protocol discovery**: Run `claude -p --output-format stream-json` with various
   prompts and tools to map the complete message schema
2. **Mock data**: Create JSON fixtures for each message type for component testing
3. **Integration test**: Full flow from input → Tauri command → binary → event → render
4. **Cross-platform**: Test on macOS (primary), Linux, Windows

## Success Criteria

- [ ] User can toggle between terminal and chat mode per session
- [ ] Chat mode renders streaming text responses with markdown
- [ ] Tool calls are displayed with collapsible details
- [ ] Permission requests show a diff and accept/reject buttons
- [ ] User can type messages and get responses in chat mode
- [ ] Chat mode works with all Claude CLI features (models, permissions, etc.)
- [ ] Both modes can coexist — some tabs terminal, some chat
- [ ] No regression in terminal mode functionality
