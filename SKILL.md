---
name: gemini-cli
description: Wield Google's Gemini CLI as a powerful auxiliary tool for code generation, review, analysis, and web research. Use when tasks benefit from a second AI perspective, current web information via Google Search, codebase architecture analysis, or parallel code generation. Also use for long-running delegation where Claude hands Gemini a multi-step job (drafting a document, staged review, iterative analysis) and sends follow-up turns to the same session by ID, optionally attaching files, images, or PDFs. Also use when user explicitly requests Gemini operations.
---

# Gemini CLI Integration Skill

This skill enables Claude Code to effectively orchestrate Gemini CLI (verified against v0.64.0) for code generation, review, analysis, and multi-turn delegation.

## Model Rule (Mandatory)

Every `gemini` invocation must pass `-m` explicitly. Only two model IDs are permitted:

- `gemini-4-argon` - the default, use it for everything
- `gemini-3.8-flash` - fallback only, when `gemini-4-argon` is rate-limited or unavailable

Never omit `-m` and never use any other model ID.

## When to Use This Skill

### Ideal Use Cases

1. **Second Opinion / Cross-Validation**
   - Code review after writing code (different AI perspective)
   - Security audit with alternative analysis
   - Finding bugs Claude might have missed

2. **Google Search Grounding**
   - Questions requiring current internet information
   - Latest library versions, API changes, documentation updates
   - Current events or recent releases

3. **Codebase Architecture Analysis**
   - Use Gemini's `codebase_investigator` tool
   - Understanding unfamiliar codebases
   - Mapping cross-file dependencies

4. **Parallel Processing**
   - Offload tasks while continuing other work
   - Run multiple code generations simultaneously
   - Background documentation generation

5. **Specialized Generation**
   - Test suite generation
   - JSDoc/documentation generation
   - Code translation between languages

6. **Long-Running Delegation**
   - Hand off a multi-step job and follow up over several turns
   - Long-form writing: drafts, docs, reports, release notes
   - Staged review where each round builds on the last
   - See "Long-Running Collaboration" below - this is the highest-value pattern

### When NOT to Use

- Simple, quick tasks (overhead not worth it)
- Tasks requiring immediate response (rate limits cause delays)
- When context is already loaded and understood

## Core Instructions

### 1. Verify Installation

```bash
command -v gemini || which gemini
```

### 2. Basic Command Pattern

```bash
gemini "[prompt]" --yolo -m gemini-4-argon -o text 2>&1
```

Key flags:

- `--yolo` or `-y`: Auto-approve all tool calls
- `-m gemini-4-argon`: Required on every call (see Model Rule above)
- `-m gemini-3.8-flash`: The only alternative, a fallback when argon is rate-limited or unavailable
- `-o text`: Human-readable output
- `-o json`: Structured output with stats

### 3. Critical Behavioral Notes

**YOLO Mode Behavior**: Auto-approves tool calls but does NOT prevent planning prompts. Gemini may still present plans and ask "Does this plan look good?" Use forceful language:

- "Apply now"
- "Start immediately"
- "Do this without asking for confirmation"

**Rate Limits**: Free tier has 60 requests/min, 1000/day. CLI auto-retries with backoff. Expect messages like "quota will reset after Xs".

**Untrusted Directories**: In a directory Gemini has not trusted, headless runs refuse and exit without answering. Pass `--skip-trust` (or set `GEMINI_CLI_TRUST_WORKSPACE=true`) for any scripted call outside a trusted project.

### 4. Output Processing

For JSON output (`-m gemini-4-argon -o json`), parse:

```json
{
  "response": "actual content",
  "session_id": "1b5fa7cb-0a7f-46d5-a0ee-040de3f00174",
  "stats": {
    "models": { "tokens": {...} },
    "tools": { "byName": {...} }
  }
}
```

Keep `session_id` whenever the task might need a follow-up. It is the handle for every later turn.

## Long-Running Collaboration (Sessions)

Every headless run creates a durable session. Use this to hand Gemini a multi-step job and send follow-ups instead of re-sending context. Prefer this over one-shot calls for anything iterative: long-form writing, staged reviews, multi-file refactors.

### 1. Open a session and hold its ID

Generate the UUID yourself so nothing needs parsing:

```bash
SID=$(uuidgen | tr 'A-Z' 'a-z')
gemini -p "Draft [thing] from @outline.md. Return the draft only." \
  -m gemini-4-argon --session-id "$SID" -o json | jq -r '.response'
```

Or let Gemini assign one and read it back from the JSON:

```bash
SID=$(gemini -p "[task]" -m gemini-4-argon -o json | jq -r '.session_id')
```

### 2. Follow up with `-r`

```bash
echo "Tighten section 2 to 400 words. Keep the opening." \
  | gemini -r "$SID" -m gemini-4-argon -o json | jq -r '.response'
```

Do not reuse `--session-id` for the follow-up. It only starts new sessions and fails with `Session ID "..." already exists. Use --resume to resume it, or provide a different ID.`.

### 3. Recover a lost ID

```bash
gemini --list-sessions   # prints "1. First prompt (2 minutes ago) [uuid]"
```

### Session Rules

- **Resume by UUID, not index.** `-r` also takes an index or `latest`, but indexes shift as new sessions are created, so a hardcoded index silently targets the wrong conversation.
- **Verify continuity.** A resumed run echoes the same `session_id` in its JSON. Check it to confirm you continued rather than started fresh.
- **Do not restate context in follow-ups.** The session already has the history. Repeating it wastes tokens and invites contradiction.
- **Watch the token growth.** Each turn re-sends the whole history. Read `stats.models.*.tokens.total` on long threads.
- **One session per job.** Do not mix an unrelated task into a session; start a new UUID.

## Headless & Concurrent Session Reliability

When running Gemini CLI in headless environments (like concurrent multi-agent setups or CI/CD pipelines), sessions are susceptible to four distinct silent failure modes:

### 1. Silent Session Eviction
* **Symptom:** Resuming a session fails with `Error resuming session: No previous sessions found for this project`.
* **Cause:** The session store (`~/.gemini/tmp/<hash>/chats`) is scoped per-project directory. Concurrent multi-agent execution causes write races or cache pruning.
* **Mitigation:** Treat session IDs as transient caches rather than durable state handles. After each turn, write the approved code/document to disk. On resume failure, automatically fall back to a fresh session, re-attaching the source files, prompt, and the disk backup.

### 2. Working Directory Drift
* **Symptom:** Resuming a session fails, and the error lists a different subdirectory where it searched for sessions.
* **Cause:** Under `--yolo` (`-y`), Gemini's tool calls are auto-approved. A tool call that changes directory (e.g. `cd`) shifts the shell's working directory.
* **Mitigation:** Check `pwd` before concluding a session is gone. Pin the working directory on every call using a subshell: `(cd "$PROJECT_ROOT" && gemini ...)` or use absolute paths for `@` file attachments. Avoid using `--yolo` for calls that only return text.

### 3. Quiet Exit 0 with Empty Response
* **Symptom:** Command succeeds (exit `0`) and returns valid JSON, but `.response` is empty or whitespace-only.
* **Cause:** Attempting to resume an evicted/dead session, or backend truncation.
* **Mitigation:** Check response content (not just exit status) on any call whose output you keep. Parse JSON and ensure `.response` is populated. Treat empty responses as dead sessions: start a fresh session with re-anchored content.

### 4. Driving Harness Timeout
* **Symptom:** Calls are killed or detached at 120 seconds (the driving tool's default timeout).
* **Cause:** First turns that attach multiple source files or index large repositories are slow.
* **Mitigation:** Set an explicit timeout (up to 10 minutes/600,000ms) on the driving tool's shell command. Alternatively, run the call in the background and poll for completion. Prefer fewer, larger turns to avoid re-attaching files.

## Attaching Files

Reference a file with `@path` inside the prompt. This works in headless mode and covers more than text:

| Target | Behavior |
|--------|----------|
| Text file | Pulled in via the `read_file` tool |
| Image (PNG, JPG, GIF, WEBP, SVG, BMP) | Inlined as a real attachment, no tool call |
| PDF | Inlined as a real attachment, no tool call |
| `@dir/` | Expands to every file in the directory |
| Absolute path outside the cwd | Resolves fine, no extra flags |

```bash
# Give a writing task its source material up front
gemini -p "Draft release notes from @CHANGELOG.md, matching the voice in @docs/voice.md." \
  -m gemini-4-argon --session-id "$SID" -o json

# Vision works
gemini -p "Transcribe the whiteboard in @~/Desktop/board.png as a markdown outline." \
  -m gemini-4-argon -o text
```

Notes:

- Images and PDFs consume prompt tokens directly since they are attached rather than read.
- Binary files do not error. You get metadata only (size, "binary data"), not contents.
- Attach source material on the session's **first** call. It stays in history for every follow-up, so later turns can just say "revise section 3".
- `.geminiignore` excludes files from `@` expansion.
- Piping is the alternative and needs no `@`: `cat notes.md | gemini -p "summarize the above"`. Stdin is prepended to the prompt.

## Quick Reference Commands

### Code Generation

```bash
gemini "Create [description] with [features]. Output complete file content." --yolo -m gemini-4-argon -o text
```

### Code Review

```bash
gemini "Review [file] for: 1) features, 2) bugs/security issues, 3) improvements" -m gemini-4-argon -o text
```

### Bug Fixing

```bash
gemini "Fix these bugs in [file]: [list]. Apply fixes now." --yolo -m gemini-4-argon -o text
```

### Test Generation

```bash
gemini "Generate [Jest/pytest] tests for [file]. Focus on [areas]." --yolo -m gemini-4-argon -o text
```

### Documentation

```bash
gemini "Generate JSDoc for all functions in [file]. Output as markdown." --yolo -m gemini-4-argon -o text
```

### Architecture Analysis

```bash
gemini "Use codebase_investigator to analyze this project" -m gemini-4-argon -o text
```

### Web Research

```bash
gemini "What are the latest [topic]? Use Google Search." -m gemini-4-argon -o text
```

### Fallback Model (Only When Argon Is Unavailable)

```bash
gemini "[prompt]" -m gemini-3.8-flash -o text
```

## Error Handling

### Rate Limit Exceeded

- CLI auto-retries with backoff
- Let the retry finish on `-m gemini-4-argon`; drop to `-m gemini-3.8-flash` only if argon stays rate-limited
- Run in background for long operations

### Command Failures

- Check JSON output for detailed error stats
- Verify Gemini is authenticated: `gemini --version`
- Check `~/.gemini/settings.json` for config issues

### Authentication Errors

If a call fails on credentials rather than the prompt, check which auth path is in play. Gemini CLI supports API key, OAuth, and Vertex AI; see [reference.md](reference.md#authentication) and defer to the [official docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/get-started/authentication.md).

For the Vertex AI service account setup, all four variables must be present:

```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to.json
GOOGLE_CLOUD_PROJECT=foo-bar-dev-055c
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=true
```

Common causes: `GOOGLE_APPLICATION_CREDENTIALS` pointing at a missing file, the service account missing `roles/aiplatform.user`, or a stray `GEMINI_API_KEY` taking precedence over Vertex.

### Validation After Generation

Always verify Gemini's output:

- Check for security vulnerabilities (XSS, injection)
- Test functionality matches requirements
- Review code style consistency
- Verify dependencies are appropriate

## Integration Workflow

### Standard Generate-Review-Fix Cycle

```bash
# 1. Generate
gemini "Create [code]" --yolo -m gemini-4-argon -o text

# 2. Review (Gemini reviews its own work)
gemini "Review [file] for bugs and security issues" -m gemini-4-argon -o text

# 3. Fix identified issues
gemini "Fix [issues] in [file]. Apply now." --yolo -m gemini-4-argon -o text
```

### Background Execution

For long tasks, run in background and monitor:

```bash
gemini "[long task]" --yolo -m gemini-4-argon -o text 2>&1 &
# Monitor with BashOutput tool
```

## Gemini's Unique Capabilities

These tools are available only through Gemini:

1. **google_web_search** - Real-time internet search via Google
2. **codebase_investigator** - Deep architectural analysis
3. **save_memory** - Cross-session persistent memory

## Configuration

### Project Context (Optional)

Create `.gemini/GEMINI.md` in project root for persistent context that Gemini will automatically read.

### Session Management

List sessions: `gemini --list-sessions`
Resume session: `echo "follow-up" | gemini -r "$SID" -m gemini-4-argon -o text`

See "Long-Running Collaboration" above for the full multi-turn workflow.

## See Also

- `reference.md` - Complete command and flag reference
- `templates.md` - Prompt templates for common operations
- `patterns.md` - Advanced integration patterns
- `tools.md` - Gemini's built-in tools documentation
