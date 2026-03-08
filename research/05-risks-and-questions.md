# Technical Risks & Open Questions

## High Risk

### 1. stream-json Protocol Is Not Fully Documented

**Risk**: The `--output-format stream-json` protocol is designed for Anthropic's own
use in the VS Code extension. The exact message types, fields, and edge cases are not
publicly documented in a formal specification.

**Impact**: We may encounter undocumented message types that break our parser, or miss
important messages that the VS Code extension handles.

**Mitigation**:
- Run extensive protocol discovery sessions with various prompts and tools
- Create comprehensive JSON fixtures from real CLI output
- Build a permissive parser that logs unknown message types instead of crashing
- Monitor Anthropic's Claude Code changelog for protocol changes
- Community resource: https://github.com/anthropics/claude-code/issues/733 has examples

### 2. Interactive Mode Limitations with --print

**Risk**: The `--print` flag is designed for non-interactive use (single prompt → response
→ exit). The VS Code extension may use internal flags or an undocumented protocol for
truly interactive sessions (multi-turn conversations within one process).

**Impact**: We might not be able to support multi-turn conversations in a single Claude
process, requiring a new process per message (expensive, loses context).

**Mitigation**:
- Test `--input-format stream-json` thoroughly — it should allow sending follow-up messages
  via stdin while the process is running
- Examine if `--print` combined with `--input-format stream-json` keeps the process alive
  waiting for stdin input
- If single-process multi-turn doesn't work: fall back to `--resume` with session IDs
  to continue conversations across process invocations
- Worst case: use `--continue` flag to resume the most recent session

### 3. Permission Request/Response Cycle

**Risk**: In terminal mode, Claude presents permission requests as interactive prompts
(y/n/always). In stream-json mode, these become structured messages that we must respond
to programmatically. The exact request/response format for each tool type may vary.

**Impact**: If we don't respond correctly, Claude will hang waiting for permission or
skip actions the user wanted.

**Mitigation**:
- Map all permission request formats for each tool (Bash, Edit, Write, Read, etc.)
- Implement a robust permission response handler with timeout fallback
- Start with `--permission-mode acceptEdits` for initial development to simplify
- Test edge cases: concurrent permission requests, cancelled requests

## Medium Risk

### 4. Diff Rendering Without VS Code APIs

**Risk**: The VS Code extension uses VS Code's native diff editor (`vscode.diff`) for
showing proposed file changes. This API doesn't exist in Tauri.

**Impact**: Diff rendering will be a significant development effort and may not match
the VS Code experience quality.

**Mitigation**:
- Use `react-diff-viewer-continued` — mature React diff component
- Or embed Monaco Editor (VS Code's editor component) in Tauri for native-quality diffs
- Start with a simple unified diff view, enhance later
- The Edit tool provides `old_string` and `new_string` — sufficient for diff computation

### 5. Claude Binary Path Discovery

**Risk**: Clutch currently runs `claude` via a shell command, assuming it's in PATH.
For the chat mode, we need to spawn `claude` as a direct child process with specific
flags. The binary location varies:
- Global install: `~/.claude/local/claude` or via `npm -g`
- VS Code extension: `~/.cursor/extensions/anthropic.claude-code-*/resources/native-binary/claude`
- Homebrew, asdf, other package managers

**Impact**: Chat mode may fail to find the Claude binary on some systems.

**Mitigation**:
- Use `which claude` to find the binary in PATH (same as terminal mode)
- Add a settings option to specify custom binary path
- Fall back to common install locations
- Show a helpful error with install instructions if not found

### 6. Message Ordering and Buffering

**Risk**: NDJSON messages arrive as a stream. With `--include-partial-messages`, we get
incremental text deltas that must be assembled in order. Network latency, buffering, and
concurrent tool calls may complicate ordering.

**Impact**: Messages could appear out of order or with missing content.

**Mitigation**:
- Use message IDs and sequence numbers for ordering
- Buffer partial messages and only render when complete (or render incrementally with
  a stable message ID as key)
- Handle the `content_block_start`, `content_block_delta`, `content_block_stop` lifecycle

### 7. Process Lifecycle Management

**Risk**: The Claude process may crash, hang, or be killed externally. Unlike a PTY
(which Clutch already handles), a direct child process has different failure modes.

**Impact**: Orphaned processes, zombie sessions, or unresponsive UI.

**Mitigation**:
- Implement robust process monitoring (check if alive periodically)
- Handle SIGTERM/SIGKILL gracefully
- Add "Restart" button for crashed sessions
- Clean up child processes on app quit (Tauri already handles this somewhat)
- Set `--max-budget-usd` as a safety valve

## Low Risk

### 8. Markdown Rendering Quality

**Risk**: Claude's responses contain rich markdown (code blocks, tables, lists, links).
The quality of markdown rendering affects the user experience significantly.

**Mitigation**: Use `react-markdown` with `remark-gfm` — battle-tested, handles all
GitHub-flavored markdown. Add `react-syntax-highlighter` for code blocks.

### 9. Performance with Long Conversations

**Risk**: Long conversations accumulate many messages. React re-renders of a long
message list could cause UI lag.

**Mitigation**:
- Use `react-window` or virtualized list for long conversations
- Memoize message components
- Limit history display with "load more" pagination

### 10. Cross-Platform Compatibility

**Risk**: Clutch supports macOS, Linux, and Windows. The Claude binary behavior and
shell spawning may differ across platforms.

**Mitigation**:
- Clutch's PTY manager already handles cross-platform shell detection
- Reuse the same logic for finding the Claude binary
- Test on all three platforms
- Windows needs special attention (Git Bash requirement for Claude Code)

## Open Questions

### Protocol Questions (Must Answer Before Implementation)

1. **Does `--input-format stream-json` keep the process alive for multi-turn?**
   - If yes: single process per session, send messages via stdin
   - If no: need to use `--resume` / `--session-id` for continuity

2. **What is the exact schema for permission request messages?**
   - Does it differ by tool type?
   - How does "always allow" work in stream-json mode?

3. **How does plan mode work in stream-json?**
   - Is the plan sent as a special message type?
   - How does the user approve/edit the plan?

4. **How does extended thinking appear in stream-json?**
   - Separate message type or within `content` array?
   - Can we show/hide thinking in the UI?

5. **What happens when Claude uses the Agent tool (subagents)?**
   - Are subagent messages nested within the parent stream?
   - How are concurrent subagent outputs interleaved?

6. **How does context compaction appear in the protocol?**
   - Is there a notification when compaction occurs?
   - Can we trigger it from the UI?

### Architecture Questions

7. **Should chat mode reuse the existing worktree system?**
   - Worktrees are currently tied to terminal sessions
   - Chat mode sessions should probably share the same worktree management

8. **Should we support switching modes mid-session?**
   - Terminal → Chat: would need to parse terminal history (hard)
   - Chat → Terminal: would need to reconstruct context (hard)
   - Recommendation: mode is set at session creation, not switchable mid-session

9. **How to handle the "default command" setting?**
   - Currently, Clutch runs the default command (e.g., `claude`) in a shell
   - Chat mode would need to use the same command but with different flags
   - Parse the command to extract the binary path?

### UX Questions

10. **What should the default mode be for new sessions?**
    - Terminal mode (backward compatible) or Chat mode (new default)?
    - Settings toggle for global default?

11. **Should we show a split view (terminal + chat side by side)?**
    - More complex but allows seeing raw output alongside structured UI
    - Could use the existing panel system

12. **How to handle tool output that is very large?**
    - File contents from Read tool can be thousands of lines
    - Collapsible blocks with "Show first N lines" + expand?

## Next Steps

1. **Protocol discovery sprint** — Run extensive testing with `claude -p --output-format
   stream-json --input-format stream-json` to answer protocol questions #1-6
2. **Prototype** — Build a minimal ChatView that renders text messages only
3. **Iterate** — Add tool calls, permissions, diffs incrementally
4. **Test** — Validate on macOS, Linux, Windows
