# Gemini CLI Integration Patterns

Advanced patterns for orchestrating Gemini CLI effectively from Claude Code.

## Pattern 1: Generate-Review-Fix Cycle

The most reliable pattern for quality code generation.

```bash
# Step 1: Generate code
gemini "Create [code description]" --yolo -m gemini-4-argon -o text

# Step 2: Have Gemini review its own work
gemini "Review [generated file] for bugs and security issues" -m gemini-4-argon -o text

# Step 3: Fix identified issues
gemini "Fix these issues in [file]: [list from review]. Apply now." --yolo -m gemini-4-argon -o text
```

### Why It Works
- Different "mindset" for generation vs review
- Self-correction catches common mistakes
- Security vulnerabilities often caught in review phase

### Example
```bash
# Generate
gemini "Create a user authentication module with bcrypt and JWT" --yolo -m gemini-4-argon -o text

# Review
gemini "Review auth.js for security vulnerabilities" -m gemini-4-argon -o text
# Output: "Found XSS risk, missing input validation, weak JWT secret"

# Fix
gemini "Fix in auth.js: XSS risk, add input validation, use env var for JWT secret. Apply now." --yolo -m gemini-4-argon -o text
```

## Pattern 2: JSON Output for Programmatic Processing

Use JSON output when you need to process results programmatically.

```bash
gemini "[prompt]" -m gemini-4-argon -o json 2>&1
```

### Parsing the Response

```javascript
// In Node.js or with jq
const result = JSON.parse(output);
const content = result.response;
const tokenUsage = result.stats.models["gemini-4-argon"].tokens.total;
const toolCalls = result.stats.tools.byName;
```

### Use Cases
- Extracting specific data from responses
- Monitoring token usage
- Tracking tool call success/failure
- Building automation pipelines

## Pattern 3: Background Execution

For long-running tasks, execute in background and continue working.

```bash
# Start in background
gemini "[long task]" --yolo -m gemini-4-argon -o text 2>&1 &

# Get process ID for later
echo $!

# Monitor output incrementally with BashOutput tool
```

### When to Use
- Code generation for large projects
- Documentation generation
- Running multiple Gemini tasks in parallel

### Parallel Execution
```bash
# Run multiple tasks simultaneously
gemini "Create frontend" --yolo -m gemini-4-argon -o text 2>&1 &
gemini "Create backend" --yolo -m gemini-4-argon -o text 2>&1 &
gemini "Create tests" --yolo -m gemini-4-argon -o text 2>&1 &
```

## Pattern 4: Model Selection Strategy

Two model IDs, nothing else. Always pass `-m` explicitly.

### Decision Tree

```
Is gemini-4-argon available (not rate-limited or erroring)?
├── Yes → -m gemini-4-argon    (the default for everything)
└── No  → -m gemini-3.8-flash  (fallback only)
```

### Examples
```bash
# Complex: Architecture analysis
gemini "Analyze codebase architecture" -m gemini-4-argon -o text

# Quick: Simple formatting
gemini "Format this JSON" -m gemini-4-argon -o text

# Trivial: One-liner
gemini "What is 2+2?" -m gemini-4-argon -o text
```

## Pattern 5: Rate Limit Handling

Strategies for working within rate limits.

### Approach 1: Let Auto-Retry Handle It
Default behavior - CLI retries automatically with backoff.

### Approach 2: Fall Back to Flash
```bash
# Only after argon keeps hitting rate limits
gemini "[task]" --yolo -m gemini-3.8-flash -o text
```

Switch back to `-m gemini-4-argon` once the limit clears.

### Approach 3: Batch Operations
Combine related operations into single prompts:
```bash
# Instead of multiple calls:
gemini "Create file A" --yolo -m gemini-4-argon
gemini "Create file B" --yolo -m gemini-4-argon
gemini "Create file C" --yolo -m gemini-4-argon

# Single call:
gemini "Create files A, B, and C with [specs]. Create all now." --yolo -m gemini-4-argon
```

### Approach 4: Sequential with Delays
For automated scripts, add delays:
```bash
gemini "[task 1]" --yolo -m gemini-4-argon -o text
sleep 2
gemini "[task 2]" --yolo -m gemini-4-argon -o text
```

## Pattern 6: Context Enrichment

Provide rich context for better results.

### Using File References
```bash
gemini "Based on @./package.json and @./src/index.js, suggest improvements" -m gemini-4-argon -o text
```

### Using GEMINI.md
Create project context that's automatically included:
```markdown
# .gemini/GEMINI.md

## Project Overview
This is a React app using TypeScript.

## Coding Standards
- Use functional components
- Prefer hooks over classes
- All functions need JSDoc
```

### Explicit Context in Prompt
```bash
gemini "Given this context:
- Project uses React 18 with TypeScript
- State management: Zustand
- Styling: Tailwind CSS

Create a user profile component." --yolo -m gemini-4-argon -o text
```

## Pattern 7: Validation Pipeline

Always validate Gemini's output before using.

### Validation Steps

1. **Syntax Check**
   ```bash
   # For JavaScript
   node --check generated.js

   # For TypeScript
   tsc --noEmit generated.ts
   ```

2. **Security Scan**
   - Check for innerHTML with user input (XSS)
   - Look for eval() or Function() calls
   - Verify input validation

3. **Functional Test**
   - Run any generated tests
   - Manual smoke test

4. **Style Check**
   ```bash
   eslint generated.js
   prettier --check generated.js
   ```

### Automated Validation Pattern
```bash
# Generate
gemini "Create utility functions" --yolo -m gemini-4-argon -o text

# Validate
node --check utils.js && eslint utils.js && npm test
```

## Pattern 8: Incremental Refinement

Build complex outputs in stages.

```bash
# Stage 1: Core structure
gemini "Create basic Express server with routes for /api/users" --yolo -m gemini-4-argon -o text

# Stage 2: Add feature
gemini "Add authentication middleware to the Express server in server.js" --yolo -m gemini-4-argon -o text

# Stage 3: Add another feature
gemini "Add rate limiting to the Express server in server.js" --yolo -m gemini-4-argon -o text

# Stage 4: Review all
gemini "Review server.js for issues and optimize" -m gemini-4-argon -o text
```

### Benefits
- Easier to debug issues
- Each stage validates before continuing
- Clear audit trail

## Pattern 9: Cross-Validation with Claude

Use both AIs for highest quality.

### Claude Generates, Gemini Reviews
```bash
# 1. Claude writes code (using normal Claude Code tools)
# 2. Gemini reviews
gemini "Review this code for bugs and security issues: [paste code]" -m gemini-4-argon -o text
```

### Gemini Generates, Claude Reviews
```bash
# 1. Gemini generates
gemini "Create [code]" --yolo -m gemini-4-argon -o text

# 2. Claude reviews the output (in conversation)
# "Review this code that Gemini generated..."
```

### Different Perspectives
- Claude: Strong on reasoning, following complex instructions
- Gemini: Strong on current web knowledge, codebase investigation

## Pattern 10: Session Continuity (Agent-to-Agent Threads)

Pin a session ID on the first call, then treat Gemini as a collaborator you send messages to. This is the default shape for any task that will take more than one turn.

```bash
# Open the thread with a fixed ID and all source material attached
SID=$(uuidgen | tr 'A-Z' 'a-z')
gemini -p "Read @src/ and map the authentication flow. Report structure only, no code changes." \
  -m gemini-4-argon --session-id "$SID" -o json | jq -r '.response'

# Follow up. No repeated context.
echo "Which of those paths can be reached without a valid token?" \
  | gemini -r "$SID" -m gemini-4-argon -o json | jq -r '.response'

# Keep going as long as the thread is useful
echo "Write the fix for the two you flagged. Apply now." \
  | gemini -r "$SID" --yolo -m gemini-4-argon -o json | jq -r '.response'
```

### Use Cases
- Iterative analysis where each question depends on the last answer
- Long-form writing with revision rounds
- Debugging sessions that accumulate findings
- Any handoff where re-sending context would cost more than the answer

### Rules
- Resume by UUID. Index and `latest` drift as new sessions appear.
- The JSON echoes `session_id` on every turn. Compare it to `$SID` to prove continuity.
- Never send `--session-id` twice. It starts sessions only and errors on reuse.
- Start a fresh UUID for an unrelated task instead of polluting a thread.

## Pattern 11: Long-Form Writing Handoff

Writing is the strongest fit for sessions, because revision needs memory of the draft.

```bash
SID=$(uuidgen | tr 'A-Z' 'a-z')

# Turn 1: brief plus every source, attached once
gemini -p "You are drafting [document]. Sources: @notes/interview.md, @data/results.csv,
voice and style rules in @docs/voice.md. Produce an outline first. Do not write prose yet." \
  -m gemini-4-argon --session-id "$SID" -o json | jq -r '.response'

# Turn 2: approve or redirect the outline
echo "Drop section 4, merge 2 and 3. Now draft the full piece at about 900 words." \
  | gemini -r "$SID" -m gemini-4-argon -o json | jq -r '.response'

# Turn 3+: targeted revisions, referencing the draft by section
echo "The opening buries the finding. Lead with the number and cut the first paragraph." \
  | gemini -r "$SID" -m gemini-4-argon -o json | jq -r '.response'

# Final: have it write the file itself
echo "Write the current draft to ./OUTPUT.md exactly as-is. Do it now." \
  | gemini -r "$SID" --yolo -m gemini-4-argon -o json | jq -r '.response'
```

### Why Sessions Beat One-Shot Prompts Here
- Revision instructions stay short ("cut the lede") because the draft is already in context
- Style and source material are attached once, not re-pasted per turn
- The outline-then-draft split catches structural problems before prose is written
- Each turn is a checkpoint, so you can redirect instead of regenerating from scratch

### Watch For
- History re-sends every turn. Check `stats.models.*.tokens.total` on long threads and start a fresh session with the final draft attached if it bloats.
- Ask for the artifact to be written to disk on the last turn rather than copying prose out of JSON.

## Pattern 12: Robust Headless Session Wrapper (Production Concurrency)

This pattern provides a slim, production-grade Bash wrapper that integrates the four critical session reliability mitigations (solving silent eviction, directory drift, empty JSON responses, and default tool timeouts) for any headless development task.

```bash
#!/usr/bin/env bash
# gemini-robust.sh - Slim & robust runner for concurrent headless Gemini CLI sessions.
# Mitigates: Directory drift, Silent Eviction, Empty responses, and Tool timeouts.

set -euo pipefail

PROJECT_ROOT="$(pwd)"
SID="${1:?Session UUID required}"
PROMPT="${2:?Prompt required}"
OUT_FILE="${3:-}" # Optional file backup of the durable state
TIMEOUT_MS=600000 # Explicit 10-minute timeout for harness limits

# 1. Prevent & Diagnose Directory Drift
CURRENT_DIR="$(pwd)"
if [[ "$CURRENT_DIR" != "$PROJECT_ROOT" ]]; then
    echo "Directory drift detected. Correcting $CURRENT_DIR -> $PROJECT_ROOT" >&2
    cd "$PROJECT_ROOT"
fi

# Run command inside a subshell to isolate internal directory drift
run_gemini() {
    local is_resume="$1"
    (
        cd "$PROJECT_ROOT"
        if [ "$is_resume" = "true" ]; then
            echo "$PROMPT" | gemini --skip-trust --timeout "$TIMEOUT_MS" -m gemini-4-argon -o json -r "$SID"
        else
            # Re-attach local backup file if it exists to recover state seamlessly
            local final_prompt="$PROMPT"
            if [ -n "$OUT_FILE" ] && [ -f "$OUT_FILE" ]; then
                final_prompt="[RECOVERY NOTE: Continuing from latest local backup @$OUT_FILE] $PROMPT"
            fi
            gemini --skip-trust --timeout "$TIMEOUT_MS" -m gemini-4-argon -o json --session-id "$SID" -p "$final_prompt"
        fi
    )
}

# 2. Resumption with Auto-Fallback
OUT=""
# Quick check if session exists locally before trying to resume
if gemini --list-sessions 2>/dev/null | grep -q "$SID"; then
    echo "Resuming session $SID..." >&2
    OUT=$(run_gemini "true" 2>/dev/null) || true
fi

# If resume failed or session didn't exist, start fresh
if [ -z "$OUT" ]; then
    echo "Session $SID missing/evicted. Starting fresh..." >&2
    OUT=$(run_gemini "false")
fi

# 3. Content Validation (Prevent Exit 0 on empty output)
BODY=$(echo "$OUT" | jq -r '.response // ""' 2>/dev/null)
if [ -z "${BODY//[[:space:]]/}" ]; then
    echo "ERROR: Gemini returned an empty response. Rejecting." >&2
    exit 1
fi

# Save output to our durable state file or stdout
if [ -n "$OUT_FILE" ]; then
    echo "$BODY" > "$OUT_FILE"
    echo "Output saved to $OUT_FILE" >&2
else
    echo "$BODY"
fi
```

### Why This Works
- **Directory Pinning:** Running the command inside a subshell `(cd "$PROJECT_ROOT" && ...)` ensures no directory drift from YOLO tool calls leaks into the calling environment.
- **Durable Disk Backups (`-o`):** Backing up generated files on disk after successful turns makes session eviction trivial to recover from by simply attaching the latest file on fallback start.
- **Empty Response Protection:** Parsing the JSON response and validating it contains actual content rejects silent exit-0 failures.
- **Explicit Timeout:** Prevents harness tools from prematurely timing out during slow first turns with large attachments.

## Anti-Patterns to Avoid

### Don't: Expect Immediate Execution
YOLO mode doesn't prevent planning. Gemini may still present plans.

**Do**: Use forceful language ("Apply now", "Start immediately")

### Don't: Ignore Rate Limits
Hammering the API wastes time on retries.

**Do**: Use appropriate models, batch operations

### Don't: Trust Output Blindly
Gemini can make mistakes, especially with security.

**Do**: Always validate generated code

### Don't: Over-Specify in Single Prompt
Extremely long prompts can confuse the model.

**Do**: Use incremental refinement for complex tasks

### Don't: Forget Context Limits
Even with 1M tokens, context can overflow.

**Do**: Use .geminiignore, be specific about files

### Don't: Re-Send Context Instead of Resuming
Pasting the previous answer back into a fresh call wastes tokens and lets the two copies drift apart.

**Do**: Capture `session_id` and follow up with `-r "$SID"`

### Don't: Resume by Index in a Script
Indexes renumber every time a session is created, so `-r 1` can hit an unrelated conversation.

**Do**: Always resume by UUID
