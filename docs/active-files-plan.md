# Active Files Context Feature

## Problem

Currently, when the `read` tool is called, file contents are stored directly in the `toolResult` message in the message history. This causes several issues:

1. **Stale content**: If a file is modified between reads, old content persists in context
2. **Context bloat**: Every read tool result stays in the message history forever
3. **No dedicated section**: File contents are scattered as individual tool results rather than a cohesive "files" section

## Solution

Implement an "active files" system where:

1. The `read` tool registers files in an `_activeFiles` set on `AgentSession` (via callback)
2. At the start of each turn, active files are reset (ensuring freshness per-turn)
3. During context generation (`transformContext`), active files are:
   - Freshly read from disk
   - Injected as a dedicated `<files>` section in the context
   - `read` tool result messages are filtered out of the context

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AgentSession                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ _activeFiles: Map<path, { content, mtime }>         │   │
│  │ _activeFilesDirty: boolean                          │   │
│  │ _activeFilesReset: boolean (per-turn flag)          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                           │
│  turn_start → _resetActiveFiles()                         │
│  read tool → onActiveFile(path) → _activeFiles.set()      │
│  transformContext → _injectActiveFiles()                  │
│    → read fresh content if dirty                          │
│    → inject as "activeFiles" custom message               │
│    → remove read toolResult messages                      │
│    → delegate to runner.emitContext()                     │
└─────────────────────────────────────────────────────────────┘
```

## Message Flow

### Before (current):
```
user message → read tool call → toolResult (file contents) → assistant response
                                                                    ↓
                                                        convertToLlm includes
                                                        the toolResult as-is
```

### After:
```
user message → read tool call → onActiveFile(path) → _activeFiles.set()
                                            ↓
                                    toolResult (minimal, for UI)
                                            ↓
                                    assistant response
                                            ↓
                                    transformContext:
                                      1. _resetActiveFiles() (new turn)
                                      2. Read fresh file content
                                      3. Inject as "activeFiles" message
                                      4. Filter out read toolResults
                                      5. Delegate to emitContext
                                            ↓
                                    convertToLlm:
                                      "activeFiles" → formatted <files> section
```

## File Contents Format

Active files are injected as a custom message with this text format:

```
<files>
## path/to/file1.ts
<content>
[file content here]
</content>

## path/to/file2.ts
<content>
[file content here]
</content>
</files>
```

This is:
- **Human-readable**: Clearly delineated sections
- **Machine-parseable**: The model can extract file paths and contents
- **Token-efficient**: Minimal wrapper markup
- **Fresh**: Always read from disk at context generation time

## Implementation Steps

### Step 1: Add ActiveFilesMessage type (`messages.ts`)

```typescript
export interface ActiveFilesMessage {
  role: "activeFiles";
  files: Array<{ path: string; content: string }>;
  timestamp: number;
}

declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    activeFiles: ActiveFilesMessage;
  }
}
```

Add `convertToLlm` handling:
```typescript
case "activeFiles": {
  let text = "<files>\n";
  for (const { path, content } of m.files) {
    text += `\n## ${path}\n<content>\n${content}\n</content>\n`;
  }
  text += "</files>";
  return {
    role: "user",
    content: [{ type: "text", text }],
    timestamp: m.timestamp,
  };
}
```

### Step 2: Add active files state to AgentSession (`agent-session.ts`)

New fields:
```typescript
private _activeFiles: Map<string, { content: string; mtime: number }> = new Map();
private _activeFilesDirty: boolean = true;
private _activeFilesInjected: boolean = false; // track per-transformContext call
```

New methods:
```typescript
private _resetActiveFiles(): void {
  this._activeFiles.clear();
  this._activeFilesDirty = true;
  this._activeFilesInjected = false;
}

private _injectActiveFiles(messages: AgentMessage[]): AgentMessage[] {
  if (this._activeFilesInjected) return messages;
  this._activeFilesInjected = true;

  // Read fresh content for any dirty files
  if (this._activeFilesDirty) {
    for (const [path] of this._activeFiles) {
      try {
        const stat = statSync(path);
        if (stat.isFile() && stat.mtimeMs > (this._activeFiles.get(path)?.mtime ?? 0)) {
          this._activeFiles.set(path, {
            content: readFileSync(path, "utf-8"),
            mtime: stat.mtimeMs,
          });
        }
      } catch {
        // File deleted or inaccessible, skip
      }
    }
    this._activeFilesDirty = false;
  }

  // Convert active files to AgentMessage
  const filesMsg: ActiveFilesMessage = {
    role: "activeFiles",
    files: Array.from(this._activeFiles.entries()).map(
      ([path, { content }]) => ({ path, content })
    ),
    timestamp: Date.now(),
  };

  // Filter out read toolResult messages
  const filtered = messages.filter(
    (m) => !(m.role === "toolResult" && (m as ToolResultMessage).toolName === "read")
  );

  // Insert active files message before the last user message (or at end if none)
  // Actually, inject at the END of the context so it's most recent
  filtered.push(filesMsg);

  return filtered;
}
```

### Step 3: Wire up turn_start reset

In `_emitExtensionEvent`, handle `turn_start`:
```typescript
if (event.type === "turn_start") {
  this._resetActiveFiles();
  // ... existing turn_start handling
}
```

### Step 4: Override `transformContext` in AgentSession

AgentSession currently delegates `transformContext` to the runner. We need to intercept it:

```typescript
// In AgentSession constructor or _buildRuntime:
this.agent.transformContext = async (messages, signal) => {
  let result = this._injectActiveFiles(messages);
  if (this._extensionRunner) {
    result = await this._extensionRunner.emitContext(result);
  }
  return result;
};
```

This replaces the current `transformContext` setup in `sdk.ts`.

### Step 5: Modify read tool to register files (`read.ts`)

Add `onActiveFile` callback option:
```typescript
export interface ReadToolOptions {
  autoResizeImages?: boolean;
  operations?: ReadOperations;
  /** Called when a file is read successfully. Path is absolute. */
  onActiveFile?: (path: string) => void;
}
```

In the `execute` method, after reading the file:
```typescript
// After reading file content successfully:
if (options?.onActiveFile && !mimeType) {
  // Only register text files as active files (not images)
  options.onActiveFile(absolutePath);
}
```

### Step 6: Wire up the callback in sdk.ts

When creating the read tool definition:
```typescript
const readOptions: ReadToolOptions = {
  autoResizeImages,
  onActiveFile: (path: string) => {
    agentSession._activeFiles.set(path, {
      content: "", // will be read fresh in _injectActiveFiles
      mtime: 0,
    });
    agentSession._activeFilesDirty = true;
  },
};
```

Actually, we need to be more careful. The `onActiveFile` callback needs access to the AgentSession instance. We can either:
- Pass the callback when creating the tool definition (closure over agentSession)
- Store a ref to agentSession in the tool options

### Step 7: Handle compaction

During compaction, active files need to be handled. Options:
- **Option A**: Active files are NOT persisted through compaction (they're transient per-turn)
- **Option B**: Persist active files as part of the compaction summary
- **Option C**: Re-register active files after compaction by re-reading the context

**Recommended**: Option A - active files are per-turn and don't survive compaction. The model will have the compaction summary which references the files, and can re-read them if needed. This is the simplest and most correct approach.

## Edge Cases

1. **File deleted between register and inject**: Handle gracefully in `_injectActiveFiles` (try/catch around `statSync`/`readFileSync`)
2. **File too large**: The existing truncation logic in the read tool handles this. Active files are just paths; content is read fresh and can be truncated by the model if needed
3. **Binary/non-text files**: Only register text files as active files. Images are handled separately (already via image content in tool results)
4. **Symlinks**: `statSync` follows symlinks by default, so this should work correctly
5. **Permission errors**: Caught by try/catch in `_injectActiveFiles`

## Testing Plan

1. Unit test: `_resetActiveFiles()` clears the set
2. Unit test: `_injectActiveFiles()` reads fresh content and formats correctly
3. Unit test: read tool calls `onActiveFile` with correct path
4. Integration test: active files appear in context, read toolResults are filtered
5. Integration test: file modifications are reflected in subsequent turns
6. Integration test: deleted files are handled gracefully

## Migration Notes

- **Breaking change**: `read` tool results are no longer in the message history. Extensions that inspect `toolResult` messages for the `read` tool will need to be updated.
- **Extension compatibility**: The `tool_result` event is still emitted with the full file contents, so extensions that react to `read` tool execution via events will continue to work.
- **Compaction**: Models may need to re-read files after compaction since they won't be in the context. The compaction summary should mention which files were active.
