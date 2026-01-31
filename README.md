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

- [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) — MCP server for n8n node docs, templates, and workflow management
- [Cursor IDE](https://cursor.com) — AI-powered IDE with MCP support
