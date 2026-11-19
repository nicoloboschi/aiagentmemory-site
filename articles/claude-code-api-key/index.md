# Claude Code API Key Setup (2026): ANTHROPIC_API_KEY, Billing and Login Order

> How to run Claude Code on an Anthropic API key with ANTHROPIC_API_KEY: the one-time approval prompt, which login wins when several are set, API billing vs a Pro or Max plan, and how to undo it. Claude Code auth docs read 2026-11-19.

Source: https://aiagentmemory.org/articles/claude-code-api-key/ · Updated 2026-11-19 · aiagentmemory.org


**Claude Code API key setup** means running Claude Code on an Anthropic API key instead of signing in with a claude.ai account. You set the `ANTHROPIC_API_KEY` variable, and Claude Code sends that key with each request. Anthropic's Claude Code authentication docs, read 2026-11-19, describe the steps below.

## Set up the key

1. **Get a key.** Create one in the Claude Console at platform.claude.com. The key belongs to your Console organization.
2. **Export it.** Set `ANTHROPIC_API_KEY` in your shell, for example in your shell profile. Keep the key out of shared scripts.
3. **Start Claude Code.** Run `claude` in your project folder. In interactive mode it asks once whether to use the key. Your answer is remembered.
4. **Check which login is active.** Run `/status`. It marks the login in use and the one that is not.

For the "use this key" choice later, open `/config` and use the "Use custom API key" toggle. The toggle only shows while `ANTHROPIC_API_KEY` is set. In non-interactive mode (`claude -p`), the key is always used when it is present, with no prompt.

If you want Claude Code to skip the browser login the first time, set the key before the first run and approve it when asked. The [install guide](/articles/install-claude-code/) shows the first-run steps.

## Which login wins

Claude Code uses the first credential it finds in this order:

1. Cloud provider variables (Bedrock, Vertex or Foundry), when one of them is set.
2. `ANTHROPIC_AUTH_TOKEN`, sent as a bearer token. Used for LLM gateways.
3. `ANTHROPIC_API_KEY`, sent as the `X-Api-Key` header.
4. The `apiKeyHelper` script, for keys that rotate, such as short-lived tokens from a vault.
5. `CLAUDE_CODE_OAUTH_TOKEN`, a one-year token from `claude setup-token`, meant for CI.
6. Anthropic profile credentials, when you name one.
7. Your `/login` subscription. This is the default for Pro, Max, Ultra, Team and Enterprise users.

The trap: if you have a subscription and also an old `ANTHROPIC_API_KEY` in your shell, Claude Code uses the key once you approve it. If that key belongs to a disabled or expired organization, requests fail. Run `unset ANTHROPIC_API_KEY` to fall back to the subscription.

## API key vs subscription

| | Subscription login (`/login`) | API key (`ANTHROPIC_API_KEY`) |
|---|---|---|
| Who pays | Your Pro, Max, Ultra, Team or Enterprise plan | Your Console organization, per token |
| Set up | Browser sign-in | Environment variable plus one approval |
| Cloud sessions | Used | Not used: cloud sessions always use the subscription, and the key does not override it |
| Rotating or scripted keys | Not for this | Use `apiKeyHelper` |

The docs say that a long-lived token from `claude setup-token` also uses your subscription, not an API key. It works for CI, but it can only make model requests. It does not enable Remote Control or claude.ai connectors.

## Where the key is kept

- **macOS:** the encrypted macOS Keychain. If the Keychain is locked, as it can be in an SSH session, Claude Code writes the login to `~/.claude/.credentials.json` instead.
- **Linux and Windows:** `~/.claude/.credentials.json`, with file mode `0600` on Linux.
- **Scripts:** `apiKeyHelper` in settings runs a shell script that prints the key. If it takes more than 10 seconds, Claude Code shows a warning. If it fails, requests fail after three tries.

To remove a key that `/login` saved, run `/logout`. The docs say nothing about the shell variable after that, so check your shell profile and run `unset ANTHROPIC_API_KEY` if it is still set.

## Keys in Console roles

Team admins decide who can make keys. In the Claude Console, the **Claude Code** role can only create Claude Code API keys. The **Developer** role can create any kind of key. The docs say this, read 2026-11-19.

## Memory and API keys

The API key only decides how Claude Code is billed and signed in. It does not give Claude Code a memory. Claude Code keeps notes in `CLAUDE.md` files and its auto memory, and a store that carries facts between sessions is a separate tool. [Claude Code memory](/articles/claude-code-memory/) covers the options.

One option is [Hindsight](https://github.com/vectorize-io/hindsight). Its Coding Agents package installs Claude Code memory with `npx @vectorize-io/hindsight-coding-agents install claude-code`. It covers 22 coding CLIs, and capture is automatic, with one memory bank per repo ([Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12). Hindsight's benchmark score on long histories is a reason to look at it: 64.1 on BEAM at 10M tokens, against 48.6 for Mem0's Platform. Both figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run, not independent.

The limits matter for your key bill. Hindsight's writes call an LLM to extract facts, so every saved session adds API usage on top of Claude Code's own calls. The retired per-agent Claude Code plugin is not updated any more; the Coding Agents package replaces it. Its benchmark figures do not show what the memory costs you on your own sessions, so measure that yourself.

Sources: [Claude Code authentication](https://code.claude.com/docs/en/authentication), read 2026-11-19; [Hindsight README](https://github.com/vectorize-io/hindsight); [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12; [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight).


## FAQ

### How do I use an API key with Claude Code?

Set the ANTHROPIC_API_KEY environment variable, then start claude. In interactive mode Claude Code asks once whether to use the key, and remembers your answer. The Claude Code authentication docs, read 2026-11-19, describe this.

### Which login does Claude Code use if I have both a key and a subscription?

The API key, once you approve it. Claude Code's precedence order puts ANTHROPIC_AUTH_TOKEN, then ANTHROPIC_API_KEY, above the /login subscription. Run unset ANTHROPIC_API_KEY to go back to the subscription, and check /status to see which one is active.

### Is Claude Code with an API key billed differently from a Pro or Max plan?

Yes. An API key is billed as Anthropic API usage on your Console account. A Pro, Max or Ultra plan is a subscription, and Claude Code signs in with it through /login. The authentication docs list the account types separately.

