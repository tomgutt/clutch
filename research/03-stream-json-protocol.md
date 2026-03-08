# Claude CLI stream-json Protocol

## Overview

The Claude Code CLI supports structured JSON input/output via the `--output-format`
and `--input-format` flags. This is the **key protocol** that enables building a rich
UI on top of Claude Code without terminal emulation.

## CLI Flags

```bash
# Output: structured JSON streaming (NDJSON — one JSON object per line)
--output-format stream-json

# Input: structured JSON streaming (send messages as JSON to stdin)  
--input-format stream-json

# Include partial/incremental message chunks as they arrive
--include-partial-messages

# Re-emit user messages from stdin back to stdout for acknowledgment
--replay-user-messages

# Full bidirectional example:
claude -p \
  --output-format stream-json \
  --input-format stream-json \
  --include-partial-messages \
  --replay-user-messages \
  "initial prompt here"
```

### Important Constraints

- `--output-format` and `--input-format` only work with `--print` (`-p`) mode
- `--print` is **non-interactive** by default — it prints the response and exits
- For a continuous interactive session, you must use `--input-format stream-json` to
  send follow-up messages via stdin
- The `--permission-mode` flag controls how permissions are handled:
  - `default` — asks for permission (sends permission request messages)
  - `acceptEdits` — auto-accepts file edits
  - `bypassPermissions` — skips all checks
  - `plan` — shows plan for approval first
  - `auto` — autonomous mode

## Output Message Format

Each line of output is a complete JSON object. The format follows the Anthropic
Messages API structure with extensions for tool use and agent actions.

### Message Types (Observed)

Based on analysis of CLI output and community documentation:

#### 1. System/Init Messages
```json
{
  "type": "system",
  "subtype": "init",
  "session_id": "uuid-here",
  "tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep", ...],
  "model": "claude-sonnet-4-6",
  "cwd": "/path/to/project"
}
```

#### 2. Assistant Messages (text response)
```json
{
  "type": "assistant",
  "message": {
    "id": "msg_01RK6k48tEj9HbGtSQt6GoTj",
    "type": "message",
    "role": "assistant",
    "content": [
      {
        "type": "text",
        "text": "Hello! How can I help you today?"
      }
    ],
    "model": "claude-sonnet-4-6",
    "stop_reason": "end_turn",
    "usage": {
      "input_tokens": 150,
      "output_tokens": 12
    }
  }
}
```

#### 3. Tool Use Messages
```json
{
  "type": "assistant",
  "message": {
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_01ABC123",
        "name": "Read",
        "input": {
          "file_path": "/path/to/file.ts",
          "limit": 100
        }
      }
    ]
  }
}
```

#### 4. Tool Result Messages
```json
{
  "type": "result",
  "subtype": "tool_result",
  "tool_use_id": "toolu_01ABC123",
  "content": "file contents here..."
}
```

#### 5. Permission Request Messages
```json
{
  "type": "permission",
  "tool": "Edit",
  "input": {
    "file_path": "/path/to/file.ts",
    "old_string": "original code",
    "new_string": "modified code"
  },
  "request_id": "perm_01XYZ"
}
```

#### 6. Partial/Streaming Messages (with --include-partial-messages)
```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "text_delta",
    "text": "Hello"
  }
}
```

#### 7. Completion/Summary
```json
{
  "type": "result",
  "subtype": "success",
  "cost_usd": 0.0042,
  "duration_ms": 3200,
  "duration_api_ms": 2800,
  "is_error": false,
  "num_turns": 1,
  "session_id": "uuid-here"
}
```

## Input Message Format

When using `--input-format stream-json`, send JSON objects to stdin:

#### User Message
```json
{
  "type": "user",
  "content": "Please explain this function"
}
```

#### Permission Response
```json
{
  "type": "permission_response",
  "request_id": "perm_01XYZ",
  "allowed": true
}
```

#### Abort/Cancel
```json
{
  "type": "abort"
}
```

## Extracting Messages with jq

Useful patterns for understanding the protocol:

```bash
# Get all text content
claude -p --output-format stream-json "hello" | \
  jq -r 'select(.content != null and (.content | type) == "array") | 
         .content[] | select(.type == "text") | .text'

# Aggregate all messages into an array
claude -p --output-format stream-json "hello" | jq -s '.'

# Filter tool calls only
claude -p --output-format stream-json "read file.ts" | \
  jq 'select(.message.content[]?.type == "tool_use")'
```

## Protocol Discovery TODO

The exact message types and fields need to be fully mapped by running the CLI with
various prompts and examining all output. Key areas to investigate:

- [ ] Full list of all `type` values
- [ ] Permission request/response cycle for each tool type
- [ ] How plan mode messages are structured
- [ ] How extended thinking messages appear
- [ ] Subagent/team messages
- [ ] Error and rate-limit messages
- [ ] Context compaction messages
- [ ] Session resume/continue messages
- [ ] The exact schema for `--input-format stream-json` user messages
- [ ] How `/slash-commands` are represented in the protocol
- [ ] How @-mentions/file references work in input

## How the VS Code Extension Uses This

Based on reverse-engineering the extension:

1. **Spawns** `resources/native-binary/claude` as a child process
2. **Flags** include `--output-format stream-json`, `--input-format stream-json`,
   `--include-partial-messages`, and various session/permission flags
3. **Reads stdout** line by line, parses JSON, forwards to webview via `postMessage`
4. **Writes to stdin** when user sends a message or responds to a permission prompt
5. **Manages lifecycle** — tracks session ID, handles binary crashes, reconnects

## Comparison with PTY Approach

| Aspect | PTY (Clutch current) | stream-json (Target) |
|--------|---------------------|---------------------|
| Data format | Raw bytes (ANSI escape codes) | Structured JSON |
| Rendering | xterm.js terminal emulator | Custom React components |
| User input | Raw keystrokes to PTY | JSON messages to stdin |
| Permissions | Terminal text prompts | Structured JSON requests |
| Diffs | ANSI-colored terminal output | Structured old/new strings |
| Tool calls | Terminal text | Named tool objects with inputs |
| Streaming | Character by character | Token-level deltas |
| Complexity | Low (just pipe bytes) | Higher (parse, render, respond) |
