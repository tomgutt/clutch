# VS Code Claude Code Extension Architecture Analysis

## Overview

The Claude Code VS Code extension (`anthropic.claude-code`) provides a **rich graphical
chat interface** for Claude Code, integrated into VS Code as a webview panel. This is
the UI we want to replicate in Clutch.

- **Extension ID**: `anthropic.claude-code`
- **Version analyzed**: 2.1.71 (darwin-arm64)
- **License**: "All rights reserved" by Anthropic PBC — **cannot embed their code**
- **Marketplace**: https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code
- **Docs**: https://code.claude.com/docs/en/vs-code

## Extension File Structure

```
anthropic.claude-code-2.1.71-darwin-arm64/
├── extension.js              # 766 lines (minified) — VS Code extension host code
├── package.json              # Extension manifest with commands, settings, activation
├── claude-code-settings.schema.json  # Settings JSON schema
├── README.md
├── resources/
│   ├── native-binary/
│   │   ├── claude            # 192MB native binary — the full Claude Code CLI
│   │   └── claude.zst        # Compressed backup (~40MB)
│   ├── claude-logo.png/svg   # Various logo assets
│   ├── AcceptMode.jpg        # Walkthrough screenshots
│   ├── PlanMode.jpg
│   ├── HighlightText.jpg
│   └── walkthrough/          # Onboarding walkthrough assets
└── webview/
    ├── index.js              # 4.7MB minified React bundle — the entire chat UI
    └── index.css             # 354KB — styles for the chat UI
```

## Two Modes of Operation

The extension supports two modes, controlled by the `claudeCode.useTerminal` setting:

### 1. Terminal Mode (`useTerminal: true`)
- Opens Claude Code in VS Code's integrated terminal
- Identical to running `claude` in any terminal
- Same as what Clutch currently does
- Raw PTY, no structured messages

### 2. Native UI Mode (`useTerminal: false`, **default**)
- Opens a **webview panel** in VS Code
- Rich graphical chat interface
- Structured message rendering (markdown, diffs, tool calls)
- Permission dialogs as UI components
- Plan mode with editable markdown documents
- @-mention autocomplete with file picker
- This is **the UI we want to replicate**

## How the Native UI Works

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  VS Code Extension Host (extension.js)                      │
│                                                             │
│  1. Registers webview panel (claudeVSCodePanel)             │
│  2. Spawns native Claude binary as child process            │
│     with --output-format stream-json                        │
│     and  --input-format stream-json                         │
│  3. Bridges messages between binary ↔ webview               │
│     using VS Code's postMessage API                         │
│                                                             │
│         ↕ postMessage / onDidReceiveMessage                 │
├─────────────────────────────────────────────────────────────┤
│  VS Code Webview (webview/index.js + index.css)             │
│                                                             │
│  - React application (~4.7MB minified)                      │
│  - Renders chat messages, tool calls, diffs                 │
│  - Handles user input, @-mentions, slash commands           │
│  - Permission request UI                                    │
│  - Plan mode review UI                                      │
│  - Context usage indicator                                  │
│  - Conversation history picker                              │
│                                                             │
│  Communicates with extension host via:                      │
│    acquireVsCodeApi().postMessage(msg)  // webview → host   │
│    window.addEventListener('message')   // host → webview   │
└─────────────────────────────────────────────────────────────┘
```

### Communication Flow

1. **Extension activates** on `onStartupFinished` or `onWebviewPanel:claudeVSCodePanel`
2. **Extension spawns** the native binary (`resources/native-binary/claude`) with:
   - `--output-format stream-json` for structured output
   - `--input-format stream-json` for structured input
   - Various flags for session management, permissions, model selection
3. **Binary outputs** NDJSON (newline-delimited JSON) messages to stdout
4. **Extension parses** each JSON line and forwards to the webview via `postMessage`
5. **Webview renders** each message as a React component
6. **User input** in the webview is sent back via `postMessage` → extension → binary stdin

### Key Evidence from extension.js

Analysis of the minified extension.js reveals:
- **8 occurrences** of `postMessage` — bidirectional webview communication
- **4 occurrences** of `onDidReceive` — message handlers
- **2 occurrences** of `createWebview` — panel creation
- **2 occurrences** of `useTerminal` — mode switching
- **1 occurrence** of `--output-format` — binary flag
- **3 occurrences** of `--json` — structured output

### package.json Configuration Points

Key settings from the extension manifest:

```json
{
  "activationEvents": ["onStartupFinished", "onWebviewPanel:claudeVSCodePanel"],
  "contributes": {
    "configuration": {
      "properties": {
        "claudeCode.useTerminal": {
          "type": "boolean",
          "default": false,
          "description": "Launch Claude in the terminal instead of the native UI."
        },
        "claudeCode.initialPermissionMode": {
          "enum": ["default", "acceptEdits", "plan", "bypassPermissions"]
        },
        "claudeCode.selectedModel": { "type": "string", "default": "default" },
        "claudeCode.autosave": { "type": "boolean", "default": true }
      }
    }
  }
}
```

## Rich UI Features to Replicate

### Must-Have (Core Experience)

| Feature | Description | VS Code Implementation |
|---------|-------------|----------------------|
| **Chat messages** | Markdown-rendered assistant/user messages | React components with markdown parser |
| **Tool call visualization** | Shows what tools Claude invokes (Read, Edit, Bash, etc.) | Collapsible blocks with tool name, args, output |
| **Permission prompts** | Accept/reject dialogs for file edits, bash commands | React dialog overlays |
| **Diff viewer** | Side-by-side or inline diff for proposed file changes | VS Code native diff API (we'd need a replacement) |
| **Input bar** | Text input with send button, permission mode selector | React input component |
| **Streaming text** | Real-time token-by-token text rendering | Incremental updates from stream-json |

### Nice-to-Have (Enhanced Experience)

| Feature | Description | Complexity |
|---------|-------------|-----------|
| **@-mentions** | File/folder autocomplete in input | Medium — needs file system integration |
| **Plan mode** | Editable markdown plan review | Medium — markdown editor component |
| **Slash commands** | `/model`, `/compact`, `/usage` etc. | Low — command palette |
| **Context indicator** | Shows context window usage % | Low — parse from messages |
| **Conversation history** | Resume past conversations | Medium — session persistence |
| **Checkpoints/rewind** | Undo to a previous state | High — needs file state tracking |
| **Extended thinking** | Toggle for deeper reasoning | Low — CLI flag |

### Not Feasible to Replicate

| Feature | Why |
|---------|-----|
| **VS Code native diffs** | Requires VS Code's diff editor API; use Monaco or react-diff-viewer instead |
| **Editor text selection context** | VS Code-specific; Clutch has no code editor |
| **VS Code command palette integration** | IDE-specific |

## Why We Cannot Embed the Webview Bundle

The `webview/index.js` (4.7MB) and `webview/index.css` (354KB) contain the complete
React application that renders the VS Code chat UI. While technically possible to host
in a Tauri webview, this is **not viable** because:

1. **License**: Anthropic's license states "All rights reserved" — redistribution prohibited
2. **VS Code API dependency**: The webview calls `acquireVsCodeApi()` for communication,
   which doesn't exist outside VS Code. Would need extensive mocking.
3. **Minified/obfuscated**: The 4.7MB bundle is minified with no source maps. Debugging
   or modifying it is impractical.
4. **Tight coupling**: Assumes VS Code's theming, editor state, file system APIs
5. **Update fragility**: Would break on every extension update (frequent, ~weekly)

**Conclusion**: We must build our own React chat UI from scratch, using the same
stream-json protocol that the extension uses to communicate with the Claude binary.
