## Cursor Cloud specific instructions

### Overview

NanoClaw is a personal AI assistant that routes messages from channels (WhatsApp, Telegram, etc.) to Claude agents running in isolated Docker containers. Single Node.js process, SQLite for storage, no external services besides Docker.

### Quick reference

Standard commands are in `package.json` scripts and `CLAUDE.md`. Key ones:

- `npm run dev` — run with hot reload (tsx)
- `npm run build` — compile TypeScript
- `npm test` — run vitest (201 unit tests, no external deps needed)
- `npm run typecheck` — tsc --noEmit
- `npm run format:check` / `npm run format:fix` — prettier
- `./container/build.sh` — rebuild the Docker agent container image

### Agent runtime

NanoClaw supports two agent runtimes, controlled by `AGENT_RUNTIME` in `.env`:

- `claude` (default) — Uses `@anthropic-ai/claude-agent-sdk` + Claude Code CLI. Auth via `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`. Requests routed through the credential proxy on port 3001.
- `cursor` — Uses Cursor Agent CLI (`agent`). Auth via `CURSOR_API_KEY` (passed directly to the container, no credential proxy). MCP config is written to `.cursor/mcp.json` in the group workspace.

### Gotchas

- **No channels in base codebase:** The base repo has no channels installed. Running `npm run dev` will initialize the DB, start the credential proxy on port 3001, then exit with "No channels connected". This is expected — channels are added via Claude Code skills (e.g., `/add-telegram`).
- **Docker required:** The app checks `docker info` at startup and exits if Docker is unavailable. In Cloud Agent environments, Docker must be started with `sudo dockerd` before running the app.
- **Container image must be pre-built:** Agents run inside the `nanoclaw-agent:latest` Docker image. Build it with `./container/build.sh` before attempting end-to-end testing. The image includes both Claude Code CLI and Cursor Agent CLI.
- **Pre-commit hook:** Husky runs `npm run format:fix` on commit. Ensure prettier formatting is clean before committing.
- **Native module:** `better-sqlite3` requires build tools (gcc, make). These are typically available in Cloud Agent VMs.
- **Node 22:** The `.nvmrc` specifies Node 22; `engines` field requires >=20.
