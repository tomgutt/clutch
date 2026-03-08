# Clutch Architecture Analysis

## Overview

Clutch is a **Tauri v2 desktop application** that provides a multi-session terminal UI
for Claude Code. Each session runs an independent PTY (pseudo-terminal) with xterm.js
rendering in the frontend.

- **Repo**: https://github.com/philip-zhan/clutch
- **Stars**: ~16 (as of March 2026)
- **License**: No formal license file, README references MIT
- **Last updated**: February 2026
- **Primary language**: TypeScript (frontend), Rust (backend)

## Tech Stack

### Frontend
| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 19.1.0 | UI framework |
| Vite | 7.0.4 | Build tool / dev server |
| TypeScript | ~5.8.3 | Type safety |
| xterm.js (`@xterm/xterm`) | 6.0.0 | Terminal emulator |
| Tailwind CSS | 4.1.18 | Styling (with inline-style workaround for spacing) |
| shadcn/ui (via `radix-ui`) | 1.4.3 | Component library |
| react-resizable-panels | 4.x | Resizable layout panels |
| lucide-react | 0.563.0 | Icons |
| sonner | 2.0.7 | Toast notifications |
| nanoid | 5.1.6 | Session ID generation |
| unique-names-generator | 4.7.1 | Branch name generation |

### Backend (Rust / Tauri)
| Crate | Version | Purpose |
|-------|---------|---------|
| tauri | 2.x | Desktop app framework, IPC |
| portable-pty | 0.8 | Cross-platform PTY management |
| tokio | 1.x | Async runtime |
| serde / serde_json | 1.x | Serialization |
| which | 7.x | Shell detection |
| tauri-plugin-store | 2.x | Persistent settings/worktree storage |
| tauri-plugin-dialog | 2.x | Native dialogs |
| tauri-plugin-os | 2.x | OS detection |
| tauri-plugin-updater | 2.x | Auto-updates |
| tauri-plugin-process | 2.x | Process management |

## Project Structure

```
clutch/
├── src/                          # React frontend
│   ├── App.tsx                   # Root component — orchestrates all state
│   ├── main.tsx                  # Entry point
│   ├── index.css                 # Global styles
│   ├── vite-env.d.ts
│   ├── components/
│   │   ├── AppLayout.tsx         # Resizable sidebar + content layout
│   │   ├── SessionContent.tsx    # Tab content — renders Terminal per session
│   │   ├── Terminal.tsx          # xterm.js wrapper with PTY integration
│   │   ├── TerminalSearchBar.tsx # In-terminal search (Cmd+F)
│   │   ├── Sidebar.tsx           # Session list sidebar
│   │   ├── TitleBar.tsx          # Window title bar
│   │   ├── Settings.tsx          # Settings panel
│   │   ├── Onboarding.tsx        # First-run onboarding flow
│   │   ├── NewSessionDialog.tsx  # New session dialog
│   │   ├── UpdateToast.tsx       # Auto-update toast
│   │   ├── onboarding/           # Onboarding sub-components
│   │   ├── shared/               # Shared components
│   │   └── ui/                   # shadcn/ui primitives (button, dialog, etc.)
│   ├── hooks/
│   │   ├── useSessionStore.ts    # Central state: sessions, settings, persistence
│   │   ├── useSessionHandlers.ts # Session lifecycle (create, close, restart)
│   │   ├── usePty.ts             # PTY Tauri IPC bridge (spawn, write, resize)
│   │   ├── useKeyboardShortcuts.ts # Global keyboard shortcuts
│   │   ├── usePolling.ts         # Activity/status polling
│   │   ├── useUpdater.ts         # Auto-update hook
│   │   └── useUpdateToast.ts     # Update notification hook
│   └── lib/                      # Utilities and helpers
├── src-tauri/                    # Rust backend
│   └── src/
│       ├── main.rs               # Tauri app entry point
│       ├── lib.rs                # Plugin registration, app setup
│       ├── commands.rs           # Tauri IPC command handlers
│       ├── pty.rs                # PTY lifecycle (spawn, read, write, resize)
│       ├── config.rs             # Configuration management
│       ├── git.rs                # Git worktree operations
│       ├── hooks_config.rs       # Hooks configuration
│       └── notifications.rs      # Notification sounds
├── assets/                       # Static assets (screenshots)
├── plans/                        # Saved implementation plans
├── CLAUDE.md                     # AI assistant instructions
├── CONTRIBUTING.md               # Contribution guide
├── package.json                  # Frontend dependencies
├── vite.config.ts                # Vite configuration
├── biome.json                    # Linter/formatter config
└── tsconfig.json                 # TypeScript config
```

## Core Architecture Concepts

### Session vs Worktree

This is a key architectural distinction documented in CLAUDE.md:

- **Session** — Ephemeral runtime object (nanoid ID). Represents a running PTY + terminal
  UI. Destroyed on app quit. Never persisted directly.
- **Worktree** — Persistent across restarts (nanoid ID). Represents a git worktree + branch.
  Stored in `@tauri-apps/plugin-store`. Cleaned up only when a tab is explicitly closed.
- On startup, one session is created per persisted worktree.
- The first session (main branch) has no worktree.
- Branch names use `unique-names-generator` (adjective-color-animal pattern).

### Data Flow: Terminal Rendering

```
┌─────────────────────────────────────────────────────────┐
│  React Frontend                                         │
│                                                         │
│  Terminal.tsx                                            │
│    ├── xterm.js instance (renders terminal output)      │
│    ├── FitAddon (auto-resize to container)              │
│    ├── SearchAddon (Cmd+F search)                       │
│    ├── WebLinksAddon (clickable URLs)                    │
│    └── Unicode11Addon (unicode support)                 │
│                                                         │
│  usePty.ts hook                                         │
│    ├── spawn(cols, rows, workingDir, command)            │
│    ├── write(data) → sends keystrokes to PTY            │
│    └── resize(cols, rows) → resizes PTY                 │
│         ↕ Tauri IPC (invoke + events)                   │
├─────────────────────────────────────────────────────────┤
│  Rust Backend (src-tauri)                               │
│                                                         │
│  commands.rs                                            │
│    ├── pty_spawn → creates PtyManager, spawns shell     │
│    ├── pty_write → writes to PTY stdin                  │
│    └── pty_resize → resizes PTY dimensions              │
│                                                         │
│  pty.rs (PtyManager)                                    │
│    ├── new(cols, rows) → opens PTY via portable-pty     │
│    ├── spawn_command(dir, cmd, env) → executes in PTY   │
│    ├── start_reader(app, session_id) → background       │
│    │   thread reads PTY output, emits "pty-data" events │
│    ├── write(data) → writes to PTY master               │
│    └── resize(cols, rows) → resizes PTY                 │
│                                                         │
│  Events emitted:                                        │
│    "pty-data"  → { session_id, data: String }           │
│    "pty-exit"  → { session_id }                         │
└─────────────────────────────────────────────────────────┘
```

### How a Session Spawns

1. User clicks "New Session" or presses Cmd+T
2. `useSessionHandlers.handleNewSession()` generates a nanoid, optionally creates a
   git worktree, and adds a Session object to the store
3. `SessionContent.tsx` renders a `<Terminal>` component for the new session
4. `Terminal.tsx` mounts, creates an xterm.js instance, calls `usePty.spawn()`
5. Rust backend receives `pty_spawn` IPC call, creates a `PtyManager`
6. `PtyManager::new()` opens a PTY via `portable-pty`
7. `PtyManager::spawn_command()` runs the configured command (default: `claude`) inside
   a login shell (`/bin/zsh -l -i -c "cd <dir> && claude; exec $SHELL"`)
8. `PtyManager::start_reader()` spawns a background thread that reads PTY output and
   emits `pty-data` Tauri events
9. Frontend `usePty` hook listens for `pty-data` events and writes raw bytes to xterm.js

### Key Insight for Our Work

The entire session rendering pipeline is **raw bytes through xterm.js**. There is no
message parsing, no structured data — just a terminal emulator displaying whatever the
PTY outputs. This is the fundamental thing we need to change for the rich UI mode.

## Code Quality & Conventions

From CLAUDE.md and CONTRIBUTING.md:

- Prefer shadcn components over custom ones
- Use inline styles for spacing (Tailwind v4 spacing utilities unreliable in Tauri)
- Use bun for package management
- Keep files under 200 lines
- TypeScript strict mode (`bun run check`)
- Biome for linting/formatting
