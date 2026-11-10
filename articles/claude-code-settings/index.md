# Claude Code Settings: Files, Precedence and Key Reference (2026)

> Where Claude Code settings live (user, shared project, project local, managed), which file wins, the main settings.json keys and their env variables, and how auto memory and Hindsight are switched on.

Source: https://aiagentmemory.org/articles/claude-code-settings/ · Updated 2026-11-10 · aiagentmemory.org


**Claude Code settings are JSON keys in a few files. Put a key in the file that matches who should get it, and the higher-ranked file wins when two files set the same key.** This page covers where the files are, which one wins, the keys people change most, and how memory settings fit in. Facts come from the Claude Code settings docs (code.claude.com/docs/en/settings and /settings-reference), read on 10 November 2026.

## Where do the settings files live?

| Scope | File | Who it affects |
|---|---|---|
| User | `~/.claude/settings.json` | You, in every project on this machine |
| Shared project | `.claude/settings.json` | Everyone who works in that folder. Commit it to share with the team |
| Project local | `.claude/settings.local.json` | You, in this one project. Claude Code keeps it out of git |
| Managed | `managed-settings.json`, MDM policy, or server-managed settings | Everyone your organization deploys it to |

Installing Claude Code creates none of these. `/config` writes `~/.claude/settings.json` for most options, and a "don't ask again" approval on a permission prompt writes an `allow` rule to `.claude/settings.local.json`.

## Which file wins?

From highest to lowest:

1. **Managed settings.** Nothing in your files overrides them. A few security keys are stricter: a `true` from any file still applies, for example `permissions.blockReadsOutsideWorkingDirectories`.
2. **Command line.** `claude --settings '{"model": "claude-opus-5-5"}'` changes one session and writes no file.
3. **Project local** (`.claude/settings.local.json`).
4. **Shared project** (`.claude/settings.json`).
5. **User** (`~/.claude/settings.json`).

Lists do not get replaced. If the team file and your user file both set `permissions.allow`, Claude Code uses both lists. The `model` key is the exception to "higher wins" in one way: an environment variable such as `ANTHROPIC_MODEL` overrides the file, and `--model` overrides both for a session.

## Keys people change most

The reference lists every key. These are the ones that come up most often. "Pairs with" is the environment variable that covers the same setting.

| Key | What it does | Pairs with |
|---|---|---|
| `model` | Model that new sessions start with. Unset means your account default | `ANTHROPIC_MODEL` |
| `effortLevel` | Effort for models that have no saved level | `CLAUDE_CODE_EFFORT_LEVEL` |
| `alwaysThinkingEnabled` | `false` turns extended thinking off | `MAX_THINKING_TOKENS` (`0` turns it off for one session) |
| `availableModels` | Limits which models you can pick | none |
| `outputStyle` / `language` | Response style and response language | none |
| `permissions.allow` / `ask` / `deny` | Tool uses that run without a prompt / always prompt / are blocked | none |
| `permissions.defaultMode` | Permission mode for new sessions. `auto` and `bypassPermissions` do not work from project or local files | `--permission-mode` for one session |
| `permissions.additionalDirectories` | Extra folders Claude may access | none |
| `env` | Environment variables for sessions and subprocesses | none |
| `hooks` | Commands that run at set points in a session | none |
| `sandbox.enabled` | Turns Bash sandboxing on | none |
| `autoMemoryEnabled` | Turns auto memory on or off | `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` |
| `statusLine` | Command that draws the status line | none |

The docs list many more keys, with the file each one may be set in and an example. Check that page before you assume a key exists.

## Example: one file, two rules

This goes in `~/.claude/settings.json`. It lets Claude run your lint and test commands without asking, and it stops Claude reading `.env` files:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)"
    ]
  }
}
```

Settings files are strict JSON. A `//` comment or a trailing comma is an error. Claude Code shows a Settings Error at the next start, or skips the bad entry and keeps the rest.

## How to check that a change loaded

- Save the file. Claude Code reloads user, project and local settings while it runs, so most changes need no restart. `model`, `effortLevel` and `modelSettings` are the exceptions for a running session: use `/model` or `/effort` instead.
- Run `/status` inside Claude Code. The **Setting sources** line lists the files it read.
- Run `claude doctor` to see entries it rejected.

Three common surprises from the docs:

- A key set in a committed project file does not reach teammates if it is a user-only key. The settings index names those keys.
- `permissions.allow` rules and most `env` values from a committed file wait until each teammate trusts the folder. `deny` and `ask` rules apply at once.
- An "always allow" click saves to your local file. An `ask` rule in a project or managed file still prompts you.

## Where memory fits

Claude Code's memory has its own settings, but it is not one switch:

- **CLAUDE.md** files are instructions you write. They load at the start of every session. No setting turns them off; you delete or move the file.
- **Auto memory** is the set of short notes Claude writes for itself, saved in `~/.claude/projects/<project>/memory/`. Turn it off with `autoMemoryEnabled: false` or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. The full guide is in [Claude Code memory: CLAUDE.md, auto memory and plugins](/articles/claude-code-memory/).
- **Skills** are not memory. They are fixed instructions for one job. See [Claude Code skills](/articles/claude-code-skills/).

[Hindsight](https://github.com/vectorize-io/hindsight) is an open-source memory server (MIT license, full server included) with a Claude Code integration. It stores facts across sessions, and its `reflect` call answers a question by reasoning over them. Three limits to plan for:

- **Writes cost tokens.** Saving facts calls an LLM, so a plugin that saves every turn costs more than one that saves rarely.
- **Recall returns more text.** Hindsight returns more context per recall than Mem0 by default, by Mem0's own measurement (about 4x). Set the token budget lower if sessions get crowded.
- **Overkill for small rule sets.** A few fixed rules fit in CLAUDE.md. Add Hindsight when the history is too large to read by hand.

Hindsight's own benchmark numbers are vendor-reported, not independent. The long-history lead is on BEAM 10M: Hindsight 64.1 against Mem0's 48.6, both figures from Mem0's comparison page.

## FAQ

**Where are Claude Code settings stored?** Four JSON files: user, shared project, project local, and managed. `~/.claude.json` holds Claude Code's own state and is not a settings file you edit.

**Which file wins?** Managed, then the command line, then project local, then shared project, then user. Lists are merged.

**How do I turn off auto memory?** `"autoMemoryEnabled": false` in a settings file, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in the environment.

**Why is my setting ignored?** A higher file sets it, the file is invalid JSON, or the key is one a project file cannot set. Run `/status` and `claude doctor`.


## FAQ

### Where are Claude Code settings stored?

Four JSON files: ~/.claude/settings.json (you, every project), .claude/settings.json (shared with the team in the repo), .claude/settings.local.json (you, one project, kept out of git), and managed settings set by an organization. Claude Code also keeps ~/.claude.json for its own state.

### Which Claude Code settings file wins?

Managed settings win first, then command-line --settings, then project local, then shared project, then user. A key set higher up overrides the same key lower down. Lists such as permissions.allow are merged, not replaced.

### How do I turn off Claude Code auto memory?

Set "autoMemoryEnabled": false in a settings file, or set the environment variable CLAUDE_CODE_DISABLE_AUTO_MEMORY=1. The /memory command has a toggle that saves the same key to ~/.claude/settings.json.

### Why is my Claude Code setting ignored?

Usually a higher level sets the same key, the key is one that a project file cannot set, or the JSON is invalid. Run /status to see which files loaded, and claude doctor to list rejected entries.

