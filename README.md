# Gemini CLI Skill for Claude Code

A Claude Code skill that enables effective use of Google's Gemini CLI as a powerful auxiliary tool.

## What This Skill Does

This skill teaches Claude Code how to wield Gemini CLI for:

- **Code Generation** - Create apps, components, and modules
- **Code Review** - Security audits, bug detection, improvements
- **Test Generation** - Unit tests, integration tests
- **Documentation** - JSDoc, README, API docs
- **Web Research** - Current information via Google Search
- **Architecture Analysis** - Codebase investigation and mapping

## Installation

Install with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# Install globally, for every project
npx skills add metaskills/gemini_cli_skill --global

# Or install into the current project only
npx skills add metaskills/gemini_cli_skill
```

Useful flags:

```bash
# Target specific agents (defaults to whatever it detects)
npx skills add metaskills/gemini_cli_skill -g -a claude-code

# Copy the files instead of symlinking
npx skills add metaskills/gemini_cli_skill -g --copy

# Pull down later updates
npx skills update gemini-cli
```

Try it without installing:

```bash
npx skills use metaskills/gemini_cli_skill
```

## Prerequisites

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed
- Authentication configured (API key, OAuth, or Vertex AI service account)

```bash
# Install Gemini CLI
npm install -g @google/gemini-cli

# Authenticate
gemini  # First run prompts for auth
```

### Authentication

There are several ways to authenticate Gemini CLI, and the [official docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/get-started/authentication.md) are the source of truth. Start there if your setup differs from what follows.

The setup I favor is Vertex AI with a service account JSON - nothing interactive, nothing user-scoped, and quota lands on a GCP project you control:

```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to.json
GOOGLE_CLOUD_PROJECT=foo-bar-dev-055c
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=true
```

This needs the Vertex AI API enabled on the project and the service account granted `roles/aiplatform.user`. Leave `GEMINI_API_KEY` unset so the CLI does not prefer it over Vertex. See [reference.md](reference.md#authentication) for the per-variable breakdown and the failure modes.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill definition - when to use, core instructions |
| `reference.md` | Complete CLI command and flag reference |
| `templates.md` | Reusable prompt templates for common tasks |
| `patterns.md` | Integration patterns and workflows |
| `tools.md` | Gemini's built-in tools documentation |

## Usage

Once installed, Claude Code automatically uses this skill when appropriate. Just ask:

```
"Use Gemini to review this code for security issues"
"Have Gemini generate tests for this module"
"Ask Gemini what's new in TypeScript 5.5"
"Get Gemini to analyze this codebase architecture"
```

## Key Features

### Prompt Templates

Ready-to-use templates for:
- Code generation (single-file, multi-file, components)
- Code review (comprehensive, security, performance)
- Test generation (unit, integration)
- Documentation (JSDoc, README, API)

### Integration Patterns

- **Session Threads** - Pin a `session_id`, then send follow-ups with `-r` instead of re-sending context
- **Long-Form Writing Handoff** - Outline, draft, revise, write to disk across turns
- **File Attachments** - `@path` for text, images, PDFs, and whole directories
- **Generate-Review-Fix** - Quality assurance cycle
- **Background Execution** - Parallel task processing
- **Model Selection** - `gemini-4-argon` by default, `gemini-3.8-flash` as a fallback only
- **Rate Limit Handling** - Strategies for free tier limits

### Gemini's Unique Tools

- `google_web_search` - Real-time internet search
- `codebase_investigator` - Deep architecture analysis
- `save_memory` - Cross-session persistence

## Quick Reference

```bash
# Basic generation
gemini "Create [description]" --yolo -m gemini-4-argon -o text

# Code review
gemini "Review [file] for bugs and security issues" -m gemini-4-argon -o text

# Web research
gemini "What's new in [topic]? Use Google Search." -m gemini-4-argon -o text

# Architecture analysis
gemini "Use codebase_investigator to analyze this project" -m gemini-4-argon -o text

# Fallback, only when argon is rate-limited or unavailable
gemini "[prompt]" -m gemini-3.8-flash -o text

# Multi-turn thread with attachments
SID=$(uuidgen | tr 'A-Z' 'a-z')
gemini -p "Draft from @notes.md" -m gemini-4-argon --session-id "$SID" -o json
echo "Tighten the opening" | gemini -r "$SID" -m gemini-4-argon -o json
```

## Why Use Gemini from Claude Code?

| Use Case | Benefit |
|----------|---------|
| Second opinion | Different AI perspective on code |
| Current info | Google Search grounding |
| Architecture | codebase_investigator tool |
| Parallel work | Offload tasks while continuing |

## License

MIT
