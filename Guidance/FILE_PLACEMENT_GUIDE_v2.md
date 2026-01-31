# File Placement Guide — n8n Workflow Builder Project (v2)

## Complete Project Structure

```
your-project-root/
│
├── CLAUDE.md                                ← Always loaded — project config & principles
│
├── .cursor/
│   ├── rules/
│   │   └── n8n-workflow-builder.mdc          ← Agent-triggered — n8n domain expertise
│   └── mcp.json                              ← ⚠️ YOUR COPY ONLY — gitignored (has secrets)
│
├── mcp.json.example                          ← ✅ Committed — template without secrets
│
├── .env                                      ← ⚠️ YOUR COPY ONLY — gitignored (has secrets)
├── .env.example                              ← ✅ Committed — template without secrets
│
├── .gitignore                                ← Git exclusion rules
├── .cursorignore                             ← Cursor AI exclusion rules
│
├── references/                               ← On-demand deep reference docs
│   ├── WORKFLOW_PATTERNS.md
│   ├── ERROR_CATALOG.md
│   └── CODE_EXAMPLES.md
│
├── workflows/                                ← Your n8n workflow JSON files
│   └── (your exported workflows go here)
│
└── README.md                                 ← Human-readable project description
```

---

## The Secrets Problem & Solution

Your project has two files that contain sensitive data:

1. **`.cursor/mcp.json`** — contains your n8n API key and URL directly in the `env` block
2. **`.env`** — contains environment variables like `N8N_API_KEY`, `N8N_API_URL`

**The solution: template + gitignore pattern.** You commit a *template* (with
placeholder values) and gitignore the *real* file (with actual secrets). Anyone
who clones your repo copies the template, fills in their own keys, and is
immediately set up.

```
mcp.json.example  ──(copy + fill in secrets)──►  .cursor/mcp.json
.env.example       ──(copy + fill in secrets)──►  .env
```

---

## File-by-File Instructions

### 1. `.gitignore`

Create this at your project root. This is the most critical file for GitHub safety.

```gitignore
# === SECRETS — NEVER COMMIT ===
.env
.env.local
.env.production
.env.*.local

# Cursor MCP config (contains API keys in env blocks)
.cursor/mcp.json

# === IDE & OS ===
.DS_Store
Thumbs.db
*.swp
*.swo
*~

# === Node (if you use npm/npx for MCP servers) ===
node_modules/
package-lock.json

# === Logs ===
*.log

# === Workflow exports with embedded credentials ===
# Uncomment if your workflow JSONs ever contain credential data:
# workflows/*-credentials.json
```

### 2. `.cursorignore`

Prevents Cursor's AI from reading sensitive files and sending them to LLM context.
Uses the same syntax as `.gitignore`.

```
# Prevent Cursor AI from indexing secrets
.env
.env.*
.cursor/mcp.json

# Large or irrelevant files
node_modules/
*.log
```

### 3. `.env.example` (committed to Git)

Template showing which environment variables are needed, with placeholder values.

```env
# n8n API Connection
N8N_API_URL=https://your-n8n-instance.com/api/v1
N8N_API_KEY=your-n8n-api-key-here

# Optional: Other services you integrate with
# SLACK_BOT_TOKEN=xoxb-your-slack-bot-token
# OPENAI_API_KEY=sk-your-openai-key-here
```

### 4. `.env` (gitignored — your local copy)

Copy `.env.example` to `.env` and fill in your real values:

```bash
cp .env.example .env
# Then edit .env with your actual keys
```

### 5. `mcp.json.example` (committed to Git)

Template for the MCP server configuration. This goes at the **project root**
(not inside `.cursor/`) so it's easy to find.

```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "npx",
      "args": ["-y", "@czlonkowski/n8n-mcp@latest"],
      "env": {
        "N8N_API_URL": "https://your-n8n-instance.com/api/v1",
        "N8N_API_KEY": "your-n8n-api-key-here"
      }
    }
  }
}
```

### 6. `.cursor/mcp.json` (gitignored — your local copy)

Copy the template and fill in your real values:

```bash
mkdir -p .cursor
cp mcp.json.example .cursor/mcp.json
# Then edit .cursor/mcp.json with your actual keys
```

**Important note on Cursor MCP config locations:**

| Location | Scope | Use When |
|---|---|---|
| `~/.cursor/mcp.json` | **Global** — all projects | MCP servers you use everywhere (GitHub, general tools) |
| `.cursor/mcp.json` | **Project** — this project only | Project-specific servers like n8n-mcp |

If you already have n8n-mcp configured globally (`~/.cursor/mcp.json`), you
don't need a project-level copy. The project-level file is useful when you want
the config to travel with the repo (via the template pattern) so collaborators
can set up quickly.

### 7. `CLAUDE.md` (committed — no secrets)

Already created. Place at project root. Contains no sensitive data — safe to commit.

### 8. `.cursor/rules/n8n-workflow-builder.mdc` (committed — no secrets)

Already created. Contains no sensitive data — safe to commit.

### 9. `references/*.md` (committed — no secrets)

Already created. All three reference files are safe to commit.

### 10. `README.md` (committed)

A human-readable project description for anyone visiting the GitHub repo.
This is for people, not AI agents. Suggested content:

```markdown
# n8n Workflow Builder

AI-assisted n8n workflow automation project built in Cursor IDE.

## Setup

1. Clone this repo
2. Copy environment templates:
   ```bash
   cp .env.example .env
   cp mcp.json.example .cursor/mcp.json
   ```
3. Edit both files with your n8n API credentials
4. Open in Cursor IDE — the MCP server and rules will load automatically

## Project Structure

- `CLAUDE.md` — AI agent instructions (project config & principles)
- `.cursor/rules/` — Domain-specific AI rules (n8n expertise)
- `references/` — Deep reference documentation for patterns, errors, code
- `workflows/` — Exported n8n workflow JSON files

## Dependencies

- [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) — MCP server for n8n
- [Cursor IDE](https://cursor.com) — AI-powered IDE with MCP support
```

---

## What Gets Committed vs Gitignored

| File | Committed? | Contains Secrets? | Purpose |
|---|---|---|---|
| `CLAUDE.md` | ✅ Yes | No | Project config for AI |
| `.cursor/rules/*.mdc` | ✅ Yes | No | Domain expertise rules |
| `references/*.md` | ✅ Yes | No | Deep reference docs |
| `mcp.json.example` | ✅ Yes | No (placeholders) | MCP config template |
| `.env.example` | ✅ Yes | No (placeholders) | Environment var template |
| `.gitignore` | ✅ Yes | No | Git exclusion rules |
| `.cursorignore` | ✅ Yes | No | Cursor AI exclusion rules |
| `README.md` | ✅ Yes | No | Human-readable docs |
| `.cursor/mcp.json` | ❌ No | **Yes — API keys** | Active MCP config |
| `.env` | ❌ No | **Yes — API keys** | Active environment vars |

---

## Setup for a New Collaborator

When someone clones your repo, they run:

```bash
git clone https://github.com/your-username/n8n-workflow-builder.git
cd n8n-workflow-builder

# Create their local secret files from templates
cp .env.example .env
cp mcp.json.example .cursor/mcp.json

# Edit with their own credentials
# Then open in Cursor — everything loads automatically
```

That's it. The rules, references, and CLAUDE.md are already in the repo.
They only need to add their own API keys.
