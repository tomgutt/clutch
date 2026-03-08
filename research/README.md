# Research: Adding a VS Code-like Rich UI to Clutch

> Research conducted March 2026. This documents the feasibility of forking Clutch
> to add a toggle between the current terminal view and a rich graphical chat UI
> similar to the Claude Code VS Code extension.

## Table of Contents

- [Executive Summary](#executive-summary)
- [Clutch Architecture Analysis](./01-clutch-architecture.md)
- [VS Code Extension Architecture Analysis](./02-vscode-extension-architecture.md)
- [Claude CLI stream-json Protocol](./03-stream-json-protocol.md)
- [Implementation Plan (Approach B)](./04-implementation-plan.md)
- [Technical Risks & Open Questions](./05-risks-and-questions.md)

## Executive Summary

**Goal**: Fork Clutch and add a button that switches each session from the current
terminal-based xterm.js view to a rich graphical chat UI equivalent to what the
Claude Code VS Code extension provides.

**Verdict**: **Feasible, with significant effort.** The core mechanism exists —
Claude Code CLI supports `--output-format stream-json` and `--input-format stream-json`
for structured bidirectional communication. This is exactly what the VS Code extension
uses under the hood. Clutch's clean Tauri + React architecture makes it a good
foundation for this work.

**Recommended approach**: Build a native React chat UI (Approach B) that communicates
with the Claude binary via the stream-json protocol, rather than trying to embed the
VS Code extension's proprietary webview bundle.

### Key Findings

| Aspect | Finding |
|--------|---------|
| **Clutch stack** | Tauri v2, React 19, xterm.js, Rust PTY backend, shadcn/ui |
| **VS Code extension** | Webview (React) + native binary, communicates via stream-json |
| **CLI protocol** | `--output-format stream-json` + `--input-format stream-json` enables bidirectional structured messaging |
| **Biggest risk** | The interactive stream-json protocol is not fully documented; reverse-engineering needed |
| **Estimated complexity** | Medium-Large (new chat components, NDJSON parser, Rust command handler) |
| **Legal** | Clutch has no formal license (but README says MIT); VS Code extension is "All rights reserved" by Anthropic — do NOT embed their webview bundle |
