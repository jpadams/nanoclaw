---
name: update-agent-runner
description: Safely make changes to the agent-runner (MCP servers, tool permissions, SDK config) or the host-side container-runner (env var forwarding, mounts). Use when adding/removing MCP servers, changing allowed tools, or modifying how containers are spawned.
---

# Updating the Agent Runner

## Architecture: Two Layers, Two Builds

```
Host process (systemd)                Container (Docker image)
──────────────────────────────────    ──────────────────────────────────
src/container-runner.ts               container/agent-runner/src/index.ts
  └─ env var forwarding                 └─ MCP server config
  └─ volume mounts                      └─ allowed tools list
  └─ spawns containers                  └─ Claude Agent SDK options
         │
         ├─ builds to: dist/            builds to: /app/dist/ (in image)
         │   npm run build                  ./container/build.sh
         │
         └─ deployed by: systemctl restart nanoclaw
```

**Rule of thumb:**
- Adding/changing an **MCP server** or **tool permission** → edit `container/agent-runner/src/index.ts`, rebuild container
- Forwarding a **new env var** into the container → edit `src/container-runner.ts`, rebuild host + restart service
- Usually you need **both** (new MCP server needs the API key forwarded too)

## Change Checklist

### 1. Edit the files

**MCP server config** (`container/agent-runner/src/index.ts`):

The SDK supports two transports. Know which one your server uses:

```typescript
// Stdio — for local binaries or mcp-remote bridges
// Install the package as a real dep (not npx) to avoid cold-start downloads.
// Use createRequire to resolve the binary path portably:
//   const MCP_REMOTE_PATH = createRequire(import.meta.url).resolve('mcp-remote/dist/proxy.js');
'my-server': {
  command: 'node',
  args: [MCP_REMOTE_PATH, 'https://example.com/mcp', '--header', `Authorization:Bearer ${process.env.MY_API_KEY}`],
}

// SSE — classic two-endpoint SSE transport (GET /sse + POST /message)
// SDK 0.2.76 runtime ONLY implements 'sse'. The 'http' type (streamable-HTTP)
// exists in the TypeScript types but is silently dropped at runtime — do not use it.
// Most modern remote MCP services (e.g. NAMS) use streamable-HTTP, not classic SSE,
// so 'sse' usually won't work for them either. Use stdio + mcp-remote instead.
'my-server': {
  type: 'sse' as const,
  url: 'https://example.com/sse',
  headers: { Authorization: `Bearer ${process.env.MY_API_KEY}` },
}
```

**For remote HTTP MCP services (streamable-HTTP protocol):** use `mcp-remote` via stdio — it bridges correctly regardless of SDK version. Install it in `container/agent-runner/package.json` and reference via `createRequire(...).resolve('mcp-remote/dist/proxy.js')`.

Also add the tool pattern to the `allowedTools` array:
```typescript
'mcp__my-server__*',
```

**Env var forwarding** (`src/container-runner.ts`):

1. Add to `readEnvFile(...)` near the top alongside similar vars:
```typescript
const myEnv = readEnvFile(['MY_API_KEY']);
```

2. Pass it into the container in `buildVolumeMounts` or `buildContainerArgs`:
```typescript
const myApiKey = process.env.MY_API_KEY || myEnv['MY_API_KEY'];
if (myApiKey) {
  args.push('-e', `MY_API_KEY=${myApiKey}`);
}
```

### 2. Build and deploy

Run these in order — don't skip steps:

```bash
# If you changed container/agent-runner/src/index.ts:
./container/build.sh

# If you changed src/container-runner.ts:
npm run build
systemctl --user restart nanoclaw
```

Both changed (most common)? Run all three.

### 3. Refresh stale agent-runner-src copies

Each group has a per-group copy of `container/agent-runner/src/` at:
```
data/sessions/{group}/agent-runner-src/
```

This copy is made **once** when the group first runs and **never auto-updated**. The container mounts it at `/app/src` (read-write, so agents can self-modify). The actual code that runs is `/app/dist/index.js` from the Docker image — but the stale source confuses the agent's self-diagnostics.

After rebuilding the container, delete stale copies so they refresh on next start:

```bash
# Refresh all groups
rm -rf data/sessions/*/agent-runner-src

# Or refresh a specific group
rm -rf data/sessions/{group}/agent-runner-src
```

The directory will be re-created from current source on the group's next container start.

## Verification

Confirm env vars reach the container by asking the agent:
> "Run `printenv | grep MY_API_KEY` and show me the output"

Confirm MCP tools are available:
> "What MCP tools do you have access to?"

Check the container image has the right dist:
```bash
docker run --rm --entrypoint /bin/bash nanoclaw-agent:latest \
  -c 'grep -c "my-server" /app/dist/index.js && echo "found"'
```

## Direct REST logging alongside MCP

When an MCP service also exposes a REST API, you can log turns directly without relying on the agent to invoke MCP tools. This is useful as a guaranteed fallback — the agent may forget to call memory tools, but REST logging fires unconditionally on every result.

Pattern used for NAMS (`container/agent-runner/src/index.ts`):

1. **Helper functions** (module-level, after imports):
   - `extractUserText(prompt)` — strips XML envelope tags to get the raw user message
   - `namsCreateConversation()` — POSTs to `/v1/conversations` once per container session, returns a `conversation_id`
   - `namsAddTurn(convId, userText, assistantText)` — fire-and-forget POST to `/v1/conversations/{id}/messages/bulk`

2. **Collect assistant text** inside the `for await` loop: accumulate `text` blocks from `message.type === 'assistant'` messages into a `assistantTextParts[]` array.

3. **Fire on result**: call `namsAddTurn(...)` just before `writeOutput(...)` in the `message.type === 'result'` branch.

4. **Create conversation once** in `main()` before the query loop; pass `namsConvId` and `extractUserText(prompt)` as extra params to `runQuery`. On subsequent turns in the same session, pass the same `namsConvId` with the new prompt.

This approach is **additive** — the MCP server still provides tools the agent can actively query; the REST logging ensures every exchange is persisted regardless.

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgot `./container/build.sh` | Agent uses old MCP config | Rebuild image |
| Forgot `npm run build` | Service starts old container-runner | Rebuild host + restart |
| Forgot `systemctl restart` | New host binary not loaded | Restart service |
| Stale `agent-runner-src` | Agent reports wrong source config | Delete per-group copies |
| Used `npx some-tool` for MCP command | Cold-start delay on every conversation | Install as npm dep or use native SDK transport |
| Checked local `container/agent-runner/dist/` | That file is not what runs in Docker | Check `/app/dist/` inside container |
