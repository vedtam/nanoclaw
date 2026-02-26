# Ticket: GitHub Copilot SDK Integration

**Status:** Open
**Priority:** Medium
**Scope:** `container/agent-runner/src/index.ts` (primary), `container/agent-runner/package.json`

---

## Summary

Replace the `@anthropic-ai/claude-agent-sdk` with the `@github/copilot-sdk` as the agent runtime in NanoClaw. This swaps the underlying LLM from Claude to GPT-4.1 while keeping all surrounding infrastructure intact: IPC, MCP server, container orchestration, WhatsApp routing, and session persistence.

---

## Motivation

- Decouple NanoClaw from a single LLM provider
- GitHub Copilot SDK exposes a production-tested agent runtime (the same engine behind Copilot CLI) with comparable capabilities to the Claude Agent SDK
- GitHub authentication instead of Anthropic API key — potentially lower cost or included in existing GitHub subscriptions

---

## Current Architecture (Claude SDK)

The agent runner (`container/agent-runner/src/index.ts`) calls `query()` from `@anthropic-ai/claude-agent-sdk` as an async generator:

```typescript
import { query, HookCallback, PreCompactHookInput, PreToolUseHookInput } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({
  prompt: stream,           // AsyncIterable<SDKUserMessage>
  options: {
    cwd: '/workspace/group',
    resume: sessionId,
    resumeSessionAt: resumeAt,
    systemPrompt: { type: 'preset', preset: 'claude_code', append: globalClaudeMd },
    allowedTools: ['Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep', 'WebSearch', 'WebFetch', 'Task', ...],
    permissionMode: 'bypassPermissions',
    mcpServers: { nanoclaw: { command: 'node', args: [...], env: {...} } },
    hooks: {
      PreCompact: [createPreCompactHook(...)],
      PreToolUse:  [{ matcher: 'Bash', hooks: [createSanitizeBashHook()] }],
    },
  }
})) { ... }
```

Key capabilities used:
- `resume` / `resumeSessionAt` — session persistence across container restarts
- `mcpServers` — injects the NanoClaw MCP tool server (`send_message`, `schedule_task`, etc.)
- `hooks.PreToolUse` — sanitizes Bash env to strip `ANTHROPIC_API_KEY`
- `hooks.PreCompact` — archives full transcript before context compaction
- Built-in tools: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebSearch`, `WebFetch`, `Task`, `TodoWrite`, `Skill`, etc.
- `systemPrompt` preset `claude_code` — provides the full Claude Code agent persona and reasoning

---

## Target Architecture (Copilot SDK)

The Copilot SDK uses a session-based API with event streaming:

```typescript
import { createClient } from '@github/copilot-sdk';

const client = createClient({ auth: process.env.GITHUB_TOKEN });

// Create or resume a session
const session = sessionId
  ? await client.resumeSession(sessionId)
  : await client.createSession({ sessionId: newId, model: 'gpt-4.1' });

// Send a message and stream results
const result = await session.sendAndWait(prompt, {
  cwd: '/workspace/group',
  mcpServers: { nanoclaw: { command: 'node', args: [...], env: {...} } },
  onPreToolUse: sanitizeBashHook,
  onPostToolUse: archiveHook,   // replaces PreCompact
});
```

Copilot SDK equivalents confirmed:
| Claude SDK feature | Copilot SDK equivalent |
|--------------------|------------------------|
| `resume: sessionId` | `client.resumeSession(id)` |
| `mcpServers` option | `mcpServers` session option |
| `hooks.PreToolUse` | `onPreToolUse` handler |
| Context compaction | "Infinite sessions" / workspace checkpoints |
| `listSessions()` | `client.listSessions()` |
| `deleteSession()` | `client.deleteSession(id)` |
| Built-in file/web tools | Confirmed (file system, Git, web requests) |

---

## Implementation Plan

### 1. Dependency swap (`container/agent-runner/package.json`)

```diff
- "@anthropic-ai/claude-agent-sdk": "^0.2.34"
+ "@github/copilot-sdk": "latest"
```

Remove `ANTHROPIC_API_KEY` from container env; add `GITHUB_TOKEN`.

### 2. Rewrite agent loop (`container/agent-runner/src/index.ts`)

- Replace `query()` async generator with `createClient` + `createSession/resumeSession` + `sendAndWait`
- The `MessageStream` class (custom async iterable for multi-turn) may no longer be needed if the Copilot SDK handles multi-turn natively via session state — verify
- Map `resume` / `resumeSessionAt` logic to `client.resumeSession(sessionId)`
- Map `newSessionId` output to `session.id` returned after each turn

### 3. Port hooks

**PreToolUse (Bash sanitization):**
```typescript
// Current
createSanitizeBashHook()  →  options.hooks.PreToolUse

// Target
onPreToolUse: (tool) => {
  if (tool.name === 'Bash') { /* strip GITHUB_TOKEN from env */ }
}
```
Note: secret name changes from `ANTHROPIC_API_KEY` to `GITHUB_TOKEN`.

**PreCompact (transcript archiving):**
The current hook fires before Claude compacts the context. The Copilot SDK uses "infinite sessions" with workspace checkpoints instead. Investigate whether `onPostToolUse` or a session event provides a suitable trigger for archiving conversation history to `/workspace/group/conversations/`.

### 4. MCP server (`container/agent-runner/src/ipc-mcp-stdio.ts`)

No changes expected — the MCP server is protocol-level and model-agnostic. Verify that the `mcpServers` option signature in the Copilot SDK matches the current config shape.

### 5. System prompt

Replace `systemPrompt: { type: 'preset', preset: 'claude_code', append: globalClaudeMd }` with an explicit system prompt string. The `claude_code` preset encodes Claude Code's agent persona — this will need to be written out manually as a system prompt passed to the Copilot SDK session. The `globalClaudeMd` append (group-level memory) should still be appended.

### 6. Auth and environment

| Variable | Current | New |
|----------|---------|-----|
| `ANTHROPIC_API_KEY` | Required | Remove |
| `GITHUB_TOKEN` | — | Required (GitHub Copilot access) |

Update `container-runner.ts` env passthrough and the Bash sanitization hook target variable.

### 7. Container rebuild

After dependency and source changes:
```bash
container builder stop && container builder rm && container builder start
./container/build.sh
```

---

## Known Gaps / Out of Scope

### `Task` tool (sub-agent spawning) — TODO

The Claude SDK's `Task` tool allows the agent to spawn parallel sub-agents. There is currently no equivalent in the Copilot SDK. This is a known capability regression.

**Resolution:** Monitor the [GitHub Copilot SDK docs](https://github.com/github/copilot-sdk) and community discussions for multi-agent orchestration support. The feature has been requested upstream: [community discussion #185990](https://github.com/orgs/community/discussions/185990). When available, implement as a follow-up.

Until then, agents that rely on `Task` for parallel work will fall back to sequential execution. Document this in the group `CLAUDE.md` files.

### Copilot SDK is in technical preview

The SDK may change in breaking ways. Pin to a specific version and review changelogs before upgrading.

### Built-in tool parity

The Copilot SDK's full built-in tool list is not yet documented in detail. Confirmed: file system ops, Git, web requests. Unconfirmed: `Glob`, `Grep`, `NotebookEdit`, `TodoWrite`, `Skill`. Test each after integration and implement as custom tools via `defineTool()` if any are missing.

---

## Testing Checklist

- [ ] Agent responds to a basic message in a group
- [ ] Session resumes correctly after container restart (check `sessions-index.json`)
- [ ] MCP tools work: `send_message`, `schedule_task`, `list_tasks`
- [ ] Bash tool does not leak `GITHUB_TOKEN` to subprocesses
- [ ] Conversation archive is written to `/workspace/group/conversations/` on context compaction
- [ ] Scheduled tasks continue to fire correctly
- [ ] Multi-turn conversation maintains context within a session

---

## References

- [Copilot SDK repo](https://github.com/github/copilot-sdk)
- [Copilot SDK Node.js README](https://github.com/github/copilot-sdk/blob/main/nodejs/README.md)
- [Persisting sessions cookbook](https://github.com/github/awesome-copilot/blob/main/cookbook/copilot-sdk/nodejs/persisting-sessions.md)
- [GitHub blog announcement](https://github.blog/news-insights/company-news/build-an-agent-into-any-app-with-the-github-copilot-sdk/)
- [Technical preview changelog](https://github.blog/changelog/2026-01-14-copilot-sdk-in-technical-preview/)
- [Multi-agent feature request](https://github.com/orgs/community/discussions/185990)
