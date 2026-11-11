# Claude Code Hooks: Every Event and How to Use Them for Memory (2026)

> The Claude Code hook events (SessionStart, UserPromptSubmit, Stop, PreCompact and 28 more: 33 in all), the four hook types, exit code rules, where hooks live in settings.json, and how hooks feed memory tools such as claude-mem and Hindsight.

Source: https://aiagentmemory.org/articles/claude-code-hooks-memory/ · Updated 2026-11-11 · aiagentmemory.org


## What are Claude Code hooks?

**Claude Code hooks** are commands that run at fixed points in a session. You set them in a JSON settings file, and Claude Code runs them when the event fires, with no model choice involved. They are the only part of Claude Code's setup that runs code on its own, which makes them the main way to save or reload memory automatically.

The Claude Code hooks reference lists 33 events. The ones that matter most for memory are `SessionStart`, `UserPromptSubmit`, `Stop`, `PreCompact` and `SessionEnd`.

## Hook events and when they fire

Event names come from the [Claude Code hooks reference](https://code.claude.com/docs/en/hooks), read 2026-11-11. The table lists the events in the order the docs give them. Memory events are first.

| Event | When it fires | Can block it? |
|---|---|---|
| `SessionStart` | When a session begins or resumes | No |
| `UserPromptSubmit` | When a prompt is submitted, before Claude processes it | Yes |
| `Stop` | When Claude finishes responding | Yes (exit 2 keeps Claude working) |
| `PreCompact` | Before context compaction (matcher: `manual` or `auto`) | Yes (exit 2 blocks compaction) |
| `PostCompact` | After context compaction completes | No |
| `SessionEnd` | When a session terminates | No |
| `Setup` | With `--init-only`, or with `--init` or `--maintenance` in `-p` mode | No |
| `UserPromptExpansion` | When a typed command expands into a prompt | Yes |
| `PreToolUse` | Before a tool call runs | Yes |
| `PermissionRequest` | When a tool call needs a permission decision | Yes (exit 2 not honored) |
| `PermissionDenied` | When auto mode denies a tool call | No |
| `PostToolUse` | After a tool call succeeds | Shows stderr to Claude only |
| `PostToolUseFailure` | After a tool call fails | No |
| `PostToolBatch` | After a batch of parallel tool calls, before the next model call | Yes (stops the loop before the next model call) |
| `Notification` | When Claude Code sends a notification | No |
| `MessageDisplay` | While assistant message text is displayed | No |
| `SubagentStart` | When a subagent is spawned | No |
| `SubagentStop` | When a subagent finishes | Yes (exit 2 keeps it working) |
| `TaskCreated` | When a task is created with `TaskCreate` | Yes (rolls back the task) |
| `TaskCompleted` | When a task is marked completed | Yes |
| `StopFailure` | When the turn ends because of an API error | No |
| `TeammateIdle` | When an agent team teammate is about to go idle | Yes (keeps the teammate working) |
| `InstructionsLoaded` | When a CLAUDE.md or `.claude/rules/*.md` file loads | No |
| `ConfigChange` | When a configuration file changes during a session | Yes (except policy settings) |
| `CwdChanged` | When the working directory changes | No |
| `DirectoryAdded` | When a directory is added mid-session with `/add-dir` | No |
| `FileChanged` | When a watched file changes on disk | No |
| `WorktreeCreate` | When a worktree is being created | Yes (any non-zero exit fails it) |
| `WorktreeRemove` | When a worktree made by a `WorktreeCreate` hook is removed | Yes (any non-zero exit fails it) |
| `PreModelSwitch` | Before a requested model switch applies | Yes |
| `PostModelSwitch` | After the session's model changes | No |
| `Elicitation` | When an MCP server asks the user for input during a tool call | Yes (declines the request) |
| `ElicitationResult` | After the user answers an MCP elicitation | Yes (response becomes decline) |

## The four hook types

- **`command`**: runs a shell command. The input arrives on stdin. The result comes back through the exit code and stdout.
- **`http`**: POSTs the event's JSON to a URL. The response uses the same JSON output format.
- **`prompt`**: sends the input to a Claude model for one turn. The model returns a JSON decision. `$ARGUMENTS` holds the input.
- **`agent`**: starts a subagent that can read files before it decides. The docs mark it experimental.

There is also `mcp_tool`, which calls a tool on a configured MCP server.

## Exit codes for command hooks

- **0**: success. If stdout is one JSON object, Claude Code reads it as a decision. Otherwise plain stdout is text. On `UserPromptSubmit`, `SessionStart` and `UserPromptExpansion`, plain text on stdout is added to Claude's context.
- **2**: a blocking error on the events that can block. The stderr text is the reason shown.
- **Any other code, including 1**: a non-blocking error. The action goes ahead, and the transcript shows a hook error. The docs say exit 1 is not a policy block; use exit 2 for that.

To make a failed command or `http` hook stop the action, set `"onFailure": "block"`. The docs say this needs Claude Code v2.1.295 or later.

## Where hooks live in settings.json

Hooks sit under a `"hooks"` key. The keys are event names. Each event has matcher groups, and each group has handlers:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/save-notes.sh" }
        ]
      }
    ]
  }
}
```

The file can be `~/.claude/settings.json` (every project), `.claude/settings.json` (shared with the repo) or `.claude/settings.local.json` (yours only). Hooks also come from managed policy settings, plugin `hooks/hooks.json` files, and the frontmatter of skills and subagents. Hooks from all levels merge. Set `"disableAllHooks": true` to turn them all off.

Two fields are handy for memory work. `Stop` gets `last_assistant_message`, which the docs recommend over reading `transcript_path`, because the transcript can lag. `SessionStart` gets `source`, which is `resume` or `fork` in those cases, so a hook can tell a fresh start from a resumed one.

## How hooks support memory

Claude Code has two built-in memory files, CLAUDE.md and auto memory. Hooks fill the gap between them: they run code when things happen, so the saved state does not depend on Claude remembering to write it.

Three patterns work with the events above:

1. **Load on start.** A `SessionStart` hook reads your saved notes and prints them. Plain stdout from this event is added to Claude's context. The docs warn that text passed this way is saved in the transcript and replayed on resume, so timestamps and commit hashes can go stale. Run the hook again on resume (it fires on resume) to refresh them.
2. **Save at the end.** A `Stop` or `SessionEnd` hook writes the session's key points to a file or a memory server.
3. **Save before compaction.** A `PreCompact` hook writes what would be lost when Claude compacts the context. Its matcher says whether it was `manual` or `auto`.

Existing tools already do this. [claude-mem](https://github.com/thedotmack/claude-mem) uses lifecycle hooks (SessionStart, UserPromptSubmit, PostToolUse, Stop and SessionEnd) to capture each session and inject context back later. [Everything Claude Code](/articles/everything-claude-code-memory/) uses hooks to save and reload session state.

## Where Hindsight fits

[Hindsight](https://github.com/vectorize-io/hindsight) is an MIT-licensed memory server from Vectorize. It has a native Claude Code plugin, so you can use it without writing your own hooks. The plugin stores what a session produces and answers later questions from it. Its `reflect` call reasons over stored memories, which helps with "why did we decide X?" questions across many sessions.

Hindsight's numbers come from Vectorize and are not independent. On the BEAM benchmark for long histories, Hindsight scores 73.9 at 1M tokens against Mem0's 64.1, and 64.1 at 10M against Mem0's 48.6. Both Mem0 figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight).

The limits:

- **More text per recall.** Hindsight returns more context than Mem0 by default, about 4x by Mem0's measurement. Set a lower token budget if prompts get crowded.
- **Writes cost model calls.** Hindsight calls an LLM when it stores memories, so a Stop hook that saves every session costs more than a plain text log.
- **Overkill for short projects.** A few rules fit in CLAUDE.md. Use a memory server when the history is too long to read.

For a full comparison, see [Claude Code memory](/articles/claude-code-memory/) and [Claude Code settings](/articles/claude-code-settings/).

## Limits of hooks

- **Hooks run your code with your permissions.** Read a hook before you add it, and do not paste hooks from strangers.
- **A hook that is slow blocks the session.** Keep save hooks fast, or send their work to a background process.
- **Exit 1 does not stop anything.** Use exit 2, or `"onFailure": "block"`, when a rule has to hold.
- **CLAUDE.md is not enforcement.** The docs say it is context. For a rule that must hold, use a `PreToolUse` hook or a `permissions.deny` rule.

## Sources

- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks), read 2026-11-11 (event list, hook types, exit codes, settings locations).
- [claude-mem README](https://github.com/thedotmack/claude-mem) (lifecycle hooks it uses).
- [Hindsight README](https://github.com/vectorize-io/hindsight) and [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight) (benchmark figures, vendor-run).


## FAQ

### What are Claude Code hooks?

Claude Code hooks are commands, HTTP calls or prompts that run at fixed points in a session, such as when a session starts, a prompt is submitted, a tool is called, or the context is compacted. You set them under the "hooks" key in settings.json. The docs list 33 event names, from SessionStart to SessionEnd.

### Can a Claude Code hook save memory?

Yes. A Stop hook or SessionEnd hook can write what happened in a session to a file or a memory server, and a SessionStart hook can load it back. Tools such as claude-mem use hooks this way. A hook only runs the code you give it, so the memory is only as good as that code.

### Is a CLAUDE.md line the same as a hook?

No. CLAUDE.md is read as context, and Claude may ignore it. A hook always runs when its event fires. For a rule that must never break, Anthropic's docs point to a PreToolUse hook or a permissions.deny rule, not a line in CLAUDE.md.

