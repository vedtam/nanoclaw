````skill
---
name: add-clear
description: Add a clear_conversation MCP tool that lets the agent reset its session on user request. Archives an optional summary, writes a marker for the host, and triggers graceful container shutdown so the next message starts fresh.
---

# Add Clear Conversation Tool

This skill adds a `clear_conversation` MCP tool to the container agent and the host-side logic to detect and honour session-clear requests. When a user says "/clear", "start fresh", or "reset", the agent can wipe its conversation session while preserving `CLAUDE.md` and all workspace files.

**UX Note:** When asking the user questions, prefer using the `AskUserQuestion` tool instead of just outputting text.

## Overview of Changes

Four files are touched:

| File | What changes |
|------|-------------|
| `container/agent-runner/src/ipc-mcp-stdio.ts` | New constant, new helper, new `clear_conversation` tool |
| `src/db.ts` | New `deleteSession()` export |
| `src/index.ts` | Import `deleteSession`, add marker-detection block in `runAgent()` |
| `groups/global/CLAUDE.md` | Document usage for all agents |

No new dependencies are required.

---

## Implementation Steps

Run all steps automatically. Only pause if a code landmark cannot be found.

### Step 1: Add `IPC_INPUT_DIR` constant to MCP server

Open `container/agent-runner/src/ipc-mcp-stdio.ts`.

Find the existing IPC directory constants near the top of the file:

```typescript
const IPC_DIR = '/workspace/ipc';
const MESSAGES_DIR = path.join(IPC_DIR, 'messages');
const TASKS_DIR = path.join(IPC_DIR, 'tasks');
```

Add a new constant immediately after `TASKS_DIR`:

```typescript
const IPC_INPUT_DIR = path.join(IPC_DIR, 'input');
```

### Step 2: Add `atomicWriteFile` helper to MCP server

In the same file (`container/agent-runner/src/ipc-mcp-stdio.ts`), find the `writeIpcFile` function (it ends with `return filename;` followed by `}`). Immediately **after** the closing brace of `writeIpcFile`, add:

```typescript
function atomicWriteFile(filePath: string, content: string): void {
  fs.mkdirSync(path.dirname(filePath), { recursive: true });
  const tempPath = `${filePath}.tmp-${process.pid}-${Date.now()}`;
  fs.writeFileSync(tempPath, content);
  fs.renameSync(tempPath, filePath);
}
```

### Step 3: Register the `clear_conversation` MCP tool

In the same file, find the final tool registration — it ends with closing `);` — right **before** the comment `// Start the stdio transport`. Insert the following tool registration:

```typescript
server.tool(
  'clear_conversation',
  `Clear the current conversation session. This resets the conversation history so the next message starts a fresh session.

Use this when the user asks to clear/reset the conversation (e.g., "/clear", "start fresh", "reset").

IMPORTANT: Before calling this tool, save any critical context to CLAUDE.md so it persists across the reset. The group's CLAUDE.md file and all files in /workspace/group/ are preserved — only the conversation session is cleared.

Optionally provide a summary that will be archived to /workspace/group/conversations/.`,
  {
    summary: z.string().optional().describe('Optional summary of the conversation being cleared. Archived for reference.'),
  },
  async (args) => {
    // Archive summary if provided
    if (args.summary) {
      try {
        const conversationsDir = '/workspace/group/conversations';
        fs.mkdirSync(conversationsDir, { recursive: true });
        const date = new Date().toISOString().split('T')[0];
        const time = new Date().toISOString().replace(/[:.]/g, '-').slice(11, 19);
        const filename = `${date}-${time}-cleared.md`;
        const filePath = path.join(conversationsDir, filename);
        const content = `# Conversation Cleared\n\nDate: ${new Date().toISOString()}\n\n## Summary\n\n${args.summary}\n`;
        fs.writeFileSync(filePath, content);
      } catch {
        // Non-fatal: archiving failure should not prevent the clear
      }
    }

    // Write marker file for the host to detect after container exits
    // TODO: Add stale-marker protection by including and validating a per-run nonce/session binding on host consumption.
    const markerPath = path.join(IPC_DIR, '_clear_session');
    atomicWriteFile(markerPath, JSON.stringify({
      timestamp: new Date().toISOString(),
      summary: args.summary || null,
    }));

    // Request immediate graceful shutdown of the current runner loop.
    // Order matters: clear marker must be visible before _close is consumed.
    const closeSentinelPath = path.join(IPC_INPUT_DIR, '_close');
    atomicWriteFile(closeSentinelPath, '');

    return {
      content: [{
        type: 'text' as const,
        text: 'Conversation session will be cleared when this response completes. The next message will start a fresh session. Your CLAUDE.md and group files are preserved.',
      }],
    };
  },
);
```

### Step 4: Add `deleteSession` to the database layer

Open `src/db.ts`.

Find the `setSession` function (it ends with `).run(groupFolder, sessionId);` followed by `}`). Immediately **after** its closing brace, add a new exported function:

```typescript
export function deleteSession(groupFolder: string): void {
  db.prepare('DELETE FROM sessions WHERE group_folder = ?').run(groupFolder);
}
```

This must appear before the existing `getAllSessions` function.

### Step 5: Import `deleteSession` in the host orchestrator

Open `src/index.ts`.

Find the import block from `'./db.js'` (or `'./db'`). It will contain imports like `getAllChats`, `getAllRegisteredGroups`, `getAllSessions`, `setSession`, etc. Add `deleteSession` to this import block. For example, change:

```typescript
import {
  getAllChats,
  getAllRegisteredGroups,
  ...
```

to:

```typescript
import {
  deleteSession,
  getAllChats,
  getAllRegisteredGroups,
  ...
```

### Step 6: Handle the clear-session marker in `runAgent()`

In `src/index.ts`, locate the `runAgent()` function. Inside it, find the block that saves the new session ID:

```typescript
    if (output.newSessionId) {
      sessions[group.folder] = output.newSessionId;
      setSession(group.folder, output.newSessionId);
    }

    if (output.status === 'error') {
```

**Between** the session-save `}` and the `if (output.status === 'error')` check, insert the clear-session marker detection logic:

```typescript
    // Check for clear-session marker written by the agent's clear_conversation tool.
    // Runs AFTER the final session save above to avoid race conditions.
    const clearMarkerPath = path.join(DATA_DIR, 'ipc', group.folder, '_clear_session');
    if (fs.existsSync(clearMarkerPath)) {
      try {
        const markerData = JSON.parse(fs.readFileSync(clearMarkerPath, 'utf-8'));
        logger.info(
          { group: group.name, summary: markerData.summary?.slice(0, 100) },
          'Clear session marker found, deleting session',
        );
      } catch {
        logger.info({ group: group.name }, 'Clear session marker found, deleting session');
      }
      deleteSession(group.folder);
      delete sessions[group.folder];
      try { fs.unlinkSync(clearMarkerPath); } catch { /* ignore */ }
    }
```

This block:
1. Checks if the container wrote a `_clear_session` marker file
2. Logs the event (with an optional summary preview)
3. Deletes the persisted session from SQLite via `deleteSession()`
4. Removes the in-memory session entry
5. Cleans up the marker file

**Important:** The marker check must run *after* the `setSession` call so the newly saved session ID is the one being deleted (the agent requested a clear *during* this run). It must appear *before* the error-status check so the session is cleared regardless of exit status.

### Step 7: Document usage in global agent memory

Open `groups/global/CLAUDE.md`.

Find the **Memory** section. It contains a line about the `conversations/` folder:

```markdown
The `conversations/` folder contains searchable history of past conversations. Use this to recall context from previous sessions.
```

Immediately **after** that line (and before "When you learn something important:"), add:

```markdown
When the user asks to clear/reset conversation history, use `mcp__nanoclaw__clear_conversation`.
- Optionally include a short summary so the reset conversation can be archived for later reference.
- Outcome: the current conversation session is cleared, the next user message starts a fresh session, and files like `CLAUDE.md` and workspace content remain unchanged.
```

### Step 8: Patch existing group session copies

The container runner copies `container/agent-runner/src/` into `data/sessions/<group>/agent-runner-src/` **only once** — when a group's session directory is first created. After that, the container mounts the per-group copy at `/app/src`. This means existing groups will **not** see the new tool unless their copies are patched.

**Do not overwrite the session files** — agents may have added group-specific tools to their copy. Instead, apply only the three additions from Steps 1–3 to each session file.

For each file at `data/sessions/*/agent-runner-src/ipc-mcp-stdio.ts`:

1. **Check if the change is already present** (idempotency guard):
   ```bash
   grep -l "clear_conversation" data/sessions/*/agent-runner-src/ipc-mcp-stdio.ts
   ```
   Any file listed already has the tool — skip it.

2. **For each file that does NOT contain `clear_conversation`**, read the file and apply the same three targeted edits from Steps 1–3:
   - After the line `const TASKS_DIR = path.join(IPC_DIR, 'tasks');`, insert the `IPC_INPUT_DIR` constant
   - After the closing `}` of the `writeIpcFile` function, insert the `atomicWriteFile` helper
   - Before the line `// Start the stdio transport`, insert the `clear_conversation` tool registration

   The exact code to insert is identical to what is described in Steps 1–3. Apply each change only if its anchor line is found and the insertion text is not already present.

### Step 9: Build and verify

```bash
npm run build
```

The build must complete cleanly. If you have tests, also run:

```bash
npm test
```

If using Apple Container, rebuild the container image so the new MCP tool is available inside the agent:

```bash
./container/build.sh
```

If using Docker, rebuild similarly:

```bash
docker build -t nanoclaw-agent:latest container/
```

Then restart the service:

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

---

## How It Works (for reference)

1. **User sends** "/clear" or similar → agent recognises the intent
2. **Agent calls** `mcp__nanoclaw__clear_conversation` (optionally with a summary)
3. **Container MCP tool**:
   - Archives the summary to `/workspace/group/conversations/<date>-cleared.md`
   - Writes `_clear_session` marker to the IPC directory
   - Writes `_close` sentinel to IPC input, requesting graceful container shutdown
4. **Host orchestrator** (after the container exits):
   - Detects the `_clear_session` marker
   - Deletes the stored session from SQLite and in-memory cache
   - Removes the marker file
5. **Next user message** → agent starts with a completely fresh session; workspace files and CLAUDE.md remain intact

## Rollback

To undo this skill manually:

1. Remove the `clear_conversation` tool and `atomicWriteFile` helper and `IPC_INPUT_DIR` constant from `container/agent-runner/src/ipc-mcp-stdio.ts`
2. Remove `deleteSession()` from `src/db.ts`
3. Remove `deleteSession` import and the clear-marker block from `src/index.ts`
4. Remove the clear_conversation guidance lines from `groups/global/CLAUDE.md`
5. Rebuild: `npm run build && ./container/build.sh`
````
