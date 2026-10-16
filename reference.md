# Gemini CLI Command Reference

Complete reference for Gemini CLI, verified against v0.64.0

## Installation

```bash
npm install -g @google/gemini-cli
# Or without installing:
npx @google/gemini-cli
```

## Authentication

Gemini CLI supports several auth paths. Defer to the [official authentication docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/get-started/authentication.md) for the full list - this section covers the common ones.

```bash
# Option 1: API Key (Gemini API / AI Studio)
export GEMINI_API_KEY=your_key

# Option 2: OAuth (interactive, personal Google account)
gemini  # First run prompts for auth
```

### Option 3: Vertex AI with a Service Account JSON

The preferred setup when the CLI runs unattended (CI, scripts, agent orchestration) or against a GCP project with billing and quota you control. No interactive login, no user-scoped token to refresh.

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/path/to.json
export GOOGLE_CLOUD_PROJECT=foo-bar-dev-055c
export GOOGLE_CLOUD_LOCATION=global
export GOOGLE_GENAI_USE_VERTEXAI=true
```

| Variable | Purpose |
|----------|---------|
| `GOOGLE_APPLICATION_CREDENTIALS` | Absolute path to the service account key JSON |
| `GOOGLE_CLOUD_PROJECT` | GCP project ID that owns the Vertex AI quota |
| `GOOGLE_CLOUD_LOCATION` | Vertex region, or `global` for the multi-region endpoint |
| `GOOGLE_GENAI_USE_VERTEXAI` | `true` routes the CLI at Vertex AI instead of the Gemini API |

Requirements:

- The Vertex AI API enabled on the project.
- The service account granted `roles/aiplatform.user` (or narrower, if you know which calls you need).
- `GEMINI_API_KEY` unset - if it is present the CLI may prefer it over Vertex.

Verify the wiring before relying on it:

```bash
gemini -p "Reply with OK" -m gemini-4-argon -o text
```

## Command Line Flags

### Essential Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--yolo` | `-y` | Auto-approve all tool calls |
| `--prompt` | `-p` | Run headless with this prompt (explicit non-interactive form) |
| `--output-format` | `-o` | Output format: `text`, `json`, `stream-json` |
| `--skip-trust` | | Trust the current directory. Required for scripted runs in untrusted dirs |
| `--model` | `-m` | Model selection - always pass explicitly: `gemini-4-argon` (default) or `gemini-3.8-flash` (fallback) |

### Session Management

| Flag | Short | Description |
|------|-------|-------------|
| `--resume` | `-r` | Resume a session by UUID (preferred), index, or "latest" |
| `--session-id` | | Start a **new** session with a UUID you supply. Errors if it already exists |
| `--session-file` | | Load a session from a JSON file |
| `--list-sessions` | | List available sessions with their UUIDs |
| `--delete-session` | | Delete session by index |

### Execution Options

| Flag | Short | Description |
|------|-------|-------------|
| `--sandbox` | `-s` | Run in isolated sandbox |
| `--approval-mode` | | `default`, `auto_edit`, or `yolo` |
| `--timeout` | | Request timeout in ms |
| `--checkpointing` | | Enable file change snapshots |

### Context & Tools

| Flag | Description |
|------|-------------|
| `--include-directories` | Add directories to workspace |
| `--allowed-tools` | Restrict available tools |
| `--allowed-mcp-server-names` | Restrict MCP servers |

### Other Options

| Flag | Short | Description |
|------|-------|-------------|
| `--debug` | `-d` | Enable debug output |
| `--version` | `-v` | Show version |
| `--help` | `-h` | Show help |
| `--list-extensions` | `-l` | List installed extensions |
| `--prompt-interactive` | `-i` | Interactive mode with initial prompt |

## Output Formats

### Text (`-m gemini-4-argon -o text`)
```bash
gemini "prompt" -m gemini-4-argon -o text
# Returns: Human-readable response
```

### JSON (`-m gemini-4-argon -o json`)
```bash
gemini "prompt" -m gemini-4-argon -o json
```

Returns structured data:
```json
{
  "response": "The actual response content",
  "stats": {
    "models": {
      "gemini-4-argon": {
        "api": {
          "totalRequests": 3,
          "totalErrors": 0,
          "totalLatencyMs": 5000
        },
        "tokens": {
          "prompt": 1500,
          "candidates": 500,
          "total": 2000,
          "cached": 800,
          "thoughts": 150,
          "tool": 50
        }
      }
    },
    "tools": {
      "totalCalls": 2,
      "totalSuccess": 2,
      "totalFail": 0,
      "byName": {
        "google_web_search": {
          "count": 1,
          "success": 1,
          "durationMs": 3000
        }
      }
    }
  }
}
```

### Stream JSON (`-o stream-json`)
Real-time newline-delimited JSON events for monitoring long tasks.

## Model Selection

### Available Models

Only these two model IDs are allowed. Always pass `-m` explicitly - never rely on the CLI default.

| Model | Use Case |
|-------|----------|
| `gemini-4-argon` | Default for everything |
| `gemini-3.8-flash` | Fallback only, when argon is rate-limited or unavailable |

### Usage
```bash
# Default
gemini "simple task" -m gemini-4-argon -o text

# Fallback only when argon is unavailable
gemini "simple task" -m gemini-3.8-flash -o text
```

## Configuration Files

### Settings Location
Priority order (highest first):
1. `/etc/gemini-cli/settings.json` (system)
2. `~/.gemini/settings.json` (user)
3. `.gemini/settings.json` (project)

### Example Settings
```json
{
  "security": {
    "auth": {
      "selectedType": "oauth-personal"
    }
  },
  "general": {
    "previewFeatures": true,
    "vimMode": false,
    "checkpointing": true
  },
  "mcpServers": {}
}
```

### Project Context (GEMINI.md)

Create `.gemini/GEMINI.md` in project root:
```markdown
# Project Context

Project description and guidelines.

## Coding Standards
- Standards Gemini should follow

## When Making Changes
- Guidelines for modifications
```

### Ignore Files (.geminiignore)

Like `.gitignore`, excludes files from context:
```
node_modules/
dist/
*.log
.env
```

## Session Management

### List Sessions
```bash
gemini --list-sessions
```

Output:
```
Available sessions for this project (5):
  1. Create task manager (10 minutes ago) [uuid]
  2. Review code (20 minutes ago) [uuid]
  ...
```

The bracketed UUID is the durable handle. Indexes shift as new sessions are created.

### Get a Session ID

Every `-o json` run returns one at the top level:

```bash
SID=$(gemini -p "[task]" -m gemini-4-argon -o json | jq -r '.session_id')
```

Or set it yourself on the first call:

```bash
SID=$(uuidgen | tr 'A-Z' 'a-z')
gemini -p "[task]" -m gemini-4-argon --session-id "$SID" -o json
```

### Resume Session
```bash
# By UUID - always prefer this
echo "follow-up question" | gemini -r "$SID" -m gemini-4-argon -o json

# By index or latest - convenient, but fragile in scripts
echo "continue" | gemini -r latest -m gemini-4-argon -o text
```

A resumed run returns the same `session_id`, which is how you confirm the follow-up landed on the right conversation. Reusing `--session-id` instead of `-r` fails with `Session ID "..." already exists`.

## Rate Limits

### Free Tier Limits
- 60 requests per minute
- 1000 requests per day

### Rate Limit Behavior
- CLI auto-retries with exponential backoff
- Message: `"quota will reset after Xs"`
- Typical wait: 1-5 seconds

### Mitigation
1. Let auto-retry finish on `gemini-4-argon`, then fall back to `gemini-3.8-flash` if it stays limited
2. Batch operations into single prompts
3. Run long tasks in background

## Interactive Commands

In interactive mode, these slash commands are available:

| Command | Purpose |
|---------|---------|
| `/help` | Show available commands |
| `/tools` | List available tools |
| `/stats` | Show token usage |
| `/compress` | Summarize context to save tokens |
| `/restore` | Restore file checkpoints |
| `/chat save <tag>` | Save conversation |
| `/chat resume <tag>` | Resume conversation |
| `/memory show` | Display GEMINI.md context |
| `/memory refresh` | Reload context files |

## Piping & Scripting

### Pipe Input
```bash
echo "What is 2+2?" | gemini -m gemini-4-argon -o text
cat file.txt | gemini "summarize this" -m gemini-4-argon -o text
```

### File Reference Syntax
In prompts, reference files with `@`:
```bash
gemini "Review @./src/main.js for bugs" -m gemini-4-argon -o text

# Images and PDFs attach as real multimodal input
gemini "Transcribe @design/board.png as markdown" -m gemini-4-argon -o text
gemini "Summarize the findings in @report.pdf" -m gemini-4-argon -o text

# Directories expand to every file inside
gemini "Review everything in @src/handlers/" -m gemini-4-argon -o text

# Absolute paths outside the workspace resolve without extra flags
gemini "Check @/Users/me/notes/spec.md against @./src" -m gemini-4-argon -o text
```

| Target | Behavior |
|--------|----------|
| Text file | Read via the `read_file` tool |
| Image, PDF | Inlined as an attachment, no tool call, costs prompt tokens |
| Directory | Expands to all contained files |
| Binary | No error, metadata only, contents skipped |
| `.geminiignore` match | Excluded from expansion |

### Shell Command Execution
In interactive mode, prefix with `!`:
```
> !git status
```

## Keyboard Shortcuts (Interactive)

| Shortcut | Function |
|----------|----------|
| `Ctrl+L` | Clear screen |
| `Ctrl+V` | Paste from clipboard |
| `Ctrl+Y` | Toggle YOLO mode |
| `Ctrl+X` | Open in external editor |

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| "API key not found" | Set `GEMINI_API_KEY`, or configure Vertex AI (see Authentication) |
| "Could not load the default credentials" | `GOOGLE_APPLICATION_CREDENTIALS` is unset or points at a missing file |
| "Permission denied on resource project" | Service account lacks `roles/aiplatform.user`, or `GOOGLE_CLOUD_PROJECT` is wrong |
| "Rate limit exceeded" | Wait for auto-retry or stay on `gemini-4-argon` |
| "Context too large" | Use `.geminiignore` or be specific |
| "Tool call failed" | Check JSON stats for details |
| "No previous sessions found" (concurrent eviction) | Keep durable state in files on disk. Fall back to fresh session and re-attach file state on failure. |
| Resume fails due to directory drift under `--yolo` | Under `--yolo`, tool calls can change parent directory. Check `pwd`. Pin directory: `(cd "$PROJECT_ROOT" && gemini ...)`. |
| Exit 0 with empty `.response` in JSON | Check response content, not just exit code. Treat empty response as a dead session and start fresh. |
| Killed/detached at 120s by driving tool | Set explicit long timeout (up to 10m/600,000ms) on shell execution, or background and poll. |

### Debug Mode
```bash
gemini "prompt" --debug -m gemini-4-argon -o text
```

### Error Reports
Full error reports saved to:
```
/var/folders/.../gemini-client-error-*.json
```
