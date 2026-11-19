# Claude Code Best Practices (2026): CLAUDE.md, Hooks, Checks

> Anthropic's Claude Code best practices: keep CLAUDE.md short, give Claude a pass or fail check, and use hooks for rules with no exceptions. Docs read 2026-11-19.

Source: https://aiagentmemory.org/articles/claude-code-best-practices/ · Updated 2026-11-19 · aiagentmemory.org


**Claude Code best practices** are the setup steps Anthropic's docs suggest when the defaults are not enough. In short: tell Claude what it cannot see in your repo, give it a check that returns pass or fail, and enforce the rules that can never break with hooks or deny rules, not with prompt wording. The steps below come from Anthropic's [best-practices page](https://code.claude.com/docs/en/best-practices), read 2026-11-19.

## Prompt wording is not a guarantee

The docs split every instruction into two kinds. A prompt, `CLAUDE.md` or a skill is read by the model, which decides what to do. A hook, deny rule or managed setting is run by Claude Code itself, every time.

| | Non-deterministic (Claude decides) | Deterministic (Claude Code enforces) |
|---|---|---|
| What to do | Your prompt, `CLAUDE.md`, skills Claude picks up itself | Hooks, and which skills only you can start |
| What it can touch | Auto mode's classifier | Deny rules, the sandbox in strict mode, managed settings |
| When it's done | A request for evidence, `/goal`, `/code-review` | Tests, typecheck, a `Stop` hook, CI |
| How much it spends | Effort level, Claude's own pace | `--max-turns` or `--max-budget-usd` with `-p`, a spend limit you've set |
| Who has the last word | Claude's summary of what it did | Branch protection, a person who approves the merge |

Most of the time, let Claude decide. Use the deterministic kind for the few rules with zero exceptions, such as "never edit the generated client in `src/gen/`."

## Tell Claude what it cannot see

Claude reads your code first, so it already knows your file layout, conventions and test framework. Write down only what the repo does not show: a decision from a meeting, a deploy process that lives in one person's head, or the environment variable your integration tests need.

The docs suggest two questions before you write anything down. Could Claude find this by reading the code? Would it actually go and look? If both answers are yes, leave it out.

- **Keep CLAUDE.md short.** Claude loads it into every session, so every line costs attention on every task. Put in the test commands, the places where the code misleads, and the decisions that are not in the repo. The memory guide says to keep each file under 200 lines. `/init` drafts one, and `/doctor` suggests cuts.
- **Put personal preferences in `CLAUDE.local.md`** and add that file to `.gitignore`.
- **Scope rules to one folder.** A file in `.claude/rules/` with a `paths:` filter loads only when Claude works on matching files.
- **Put team procedures in skills.** A skill at `.claude/skills/<name>/SKILL.md` loads in full only when the task needs it. For a step with side effects, like a release, add `disable-model-invocation: true` so Claude cannot start it alone.
- **Connect the tools Claude needs.** If a system has a CLI, like `gh`, Claude can use it with no extra setup. For other systems, add an MCP server, for example `claude mcp add --transport http notion https://mcp.notion.com/mcp`. Connected servers add their tool names to the context, so turn off the ones you do not use.

## Give Claude a check to run

Claude stops when the work looks done. A check that returns pass or fail lets it close the loop itself: it makes the change, runs the check, reads the failure and tries again. The docs list a test file, a build's exit code, a linter, or a screenshot as good checks. The most useful line in CLAUDE.md is often the command that runs them.

There are four ways to apply the check, from loosest to strictest:

1. **In the prompt:** "run the auth tests after, and keep going until they pass."
2. **As a goal:** `/goal` makes a separate model check the condition after every turn.
3. **As a `Stop` hook:** the harness runs your script whenever Claude tries to end its turn. Exit code 2 sends Claude back to work. Here is a typecheck hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "pnpm tsc --noEmit 1>&2 || exit 2" }
        ]
      }
    ]
  }
}
```

If the hook blocks eight stops in a row with no tool call between them, Claude Code ends the turn anyway. Raise that cap with `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`.

4. **By a fresh reviewer:** `/code-review` checks the diff in a subagent that never saw the conversation.

Whichever you use, ask for evidence: the command and its output, or the screenshot. "All tests pass" alone tells you little.

## Enforce the rules with no exceptions

Hooks run your command at fixed points: before or after a tool call, when Claude tries to stop, or when a session starts. Use a `command`, `http` or `mcp_tool` hook for real enforcement. Add `"onFailure": "block"` if the action should stop when the hook itself crashes. A `prompt` or `agent` hook asks another model to decide, so it is not deterministic.

A `PostToolUse` hook can run a formatter after every edit, so Claude no longer has to remember the rule. Permission rules work the same way. A `deny` rule holds in every permission mode, including auto mode and `bypassPermissions`. Two limits from the docs:

- A `Read` deny rule catches `cat .env`, but not a script that opens the file.
- A `WebFetch` deny rule for a domain does nothing about `curl` to that domain, unless the sandbox is on.

Where a rule lives decides who it binds. Your `~/.claude/settings.json` and `.claude/settings.local.json` are yours alone. `.claude/settings.json` belongs to the repo. Managed settings belong to the organization and cannot be overridden by personal files. The docs warn to keep the enforced set small, because every hook and deny rule will sometimes fire when it should not.

## Change a default when a task costs too much

Interactive terminal and VS Code sessions start in **auto mode**. A separate classifier model checks each action first. It blocks the risky things, such as `curl | bash`, production deploys and migrations, and force pushes over other people's work. After three blocks in a row, or twenty in total, auto mode pauses and asks you.

Effort is the main control for how long Claude thinks. The docs say Opus 5.5 runs at `medium` effort on most plans, and `/effort high` or `/effort xhigh` is for work where edge cases are easy to miss. Subagents add cost. The docs suggest `model: haiku` for simple sub-jobs, and `tools: Read, Grep, Glob` for a reviewer that cannot edit files.

Cost also comes from context. Every request sends the whole conversation. Run `/clear` between unrelated tasks, and `/compact` with a focus when you keep working on one thing. The prompt cache lasts about an hour on a subscription and five minutes on an API key.

## Steer the session

- Press `Esc` to stop Claude mid-action without losing work.
- Use `/rewind` to roll back code, conversation or both.
- After two failed corrections, run `/clear` and write one better prompt. A third correction mostly adds to the failed attempts Claude re-reads.

## Let Claude run without you

Before an unattended run, the docs ask five questions: does it know what done looks like, can it check its own work, what may it touch, what stops it, and who signs off on the result. For scripts, `claude -p` with `--allowedTools`, `--permission-mode dontAsk` and `--max-turns` covers most of that. `--bare` skips hooks, skills, plugins and `CLAUDE.md` files, so do not use it for a run that needs them.

For a repeat job inside an open session, `/loop 15m check CI on this branch` runs a prompt on an interval. Each run is another turn in the same session, so do not leave one running in a long conversation.

## Stop doing these

The docs list advice that current models no longer need. Two of them:

| The old advice | Why it's gone |
|---|---|
| Add IMPORTANT or ALL CAPS to instructions Claude keeps missing | Current ones follow a plainly worded line, and piling on emphasis mostly makes Claude over-apply the rule. If it's a rule that can't bend, turn it into a hook or a deny rule. If the line is getting lost in a long file, the file needs to be shorter. |
| `think hard`, `think harder`, `be thorough` in the prompt | Effort is the control for that. Those phrases are ordinary prompt text that nudges Claude within its effort setting rather than keywords that raise it. `ultrathink` still works: it asks for deeper reasoning on that single turn without touching your effort setting. |

## Memory and Claude Code best practices

CLAUDE.md is the memory Claude Code reads at the start of each session. It is context, not enforced config, so the docs ask you to keep it short and to use hooks for anything that must hold. A store that keeps facts across sessions is a separate tool. [Claude Code memory](/articles/claude-code-memory/) compares the options.

One of them is [Hindsight](https://github.com/vectorize-io/hindsight). Its Coding Agents package installs memory for Claude Code with `npx @vectorize-io/hindsight-coding-agents install claude-code`. It covers 22 coding CLIs and saves to one memory bank per repo ([Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12). On long histories, Hindsight scored 64.1 on BEAM at 10M tokens, against 48.6 for Mem0's Platform. Both figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run, not independent.

The limit: Hindsight calls an LLM to extract facts on every save, so memory adds to your Claude Code usage. Test that cost on your own sessions before you rely on it.

Sources: [Claude Code best practices](https://code.claude.com/docs/en/best-practices), read 2026-11-19; [Claude Code memory](https://code.claude.com/docs/en/memory), read 2026-11-19; [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12; [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight).


## FAQ

### What are the main Claude Code best practices?

Tell Claude only what it cannot read from the code, give it a check it can run that returns pass or fail, and move rules that must never break into hooks or deny rules. Anthropic's best-practices page, read 2026-11-19, lists these three points first.

### How long should CLAUDE.md be?

The Claude Code memory guide says to keep each CLAUDE.md under 200 lines. Longer files use more context and Claude follows them less reliably. Move rules that only apply to one folder into .claude/rules/ with a paths filter.

### When should I use a hook instead of a CLAUDE.md line?

Use a hook when something must happen every time, with no exceptions. The docs say a CLAUDE.md line is a guideline that Claude may decide does not apply. A hook runs whether or not Claude remembers the rule.

