# Codex API Key Setup (2026): Login Command, Billing and Limits

> How to sign Codex CLI in with an OpenAI API key: the codex login --with-api-key command, where the key is stored, API billing vs ChatGPT plans, and what the key cannot do. OpenAI Codex auth docs read 2026-11-19.

Source: https://aiagentmemory.org/articles/codex-api-key/ · Updated 2026-11-19 · aiagentmemory.org


**Codex API key setup** means signing the Codex CLI in with an OpenAI API key instead of your ChatGPT account. You put the key in the `OPENAI_API_KEY` variable and run `codex login --with-api-key`. OpenAI's Codex auth page says API key use is billed at standard API rates, and that some features that need a ChatGPT workspace or cloud access are not available.

## Set up the API key

1. **Get a key.** Create one in the OpenAI dashboard. The key belongs to your OpenAI Platform account.
2. **Export it.** Set `OPENAI_API_KEY` in your shell, for example in your shell profile. Avoid pasting the key into shell history.
3. **Log in.** Run `printenv OPENAI_API_KEY | codex login --with-api-key`. The key goes in through stdin, so it does not appear in the command line.
4. **Check the login.** Start `codex` in a repo and send a short prompt. If Codex asks you to sign in, the login did not save.
5. **Pick where the login is stored.** The `cli_auth_credentials_store` setting in `config.toml` chooses `file`, `keyring`, `auto` or `ephemeral`. `auto` uses the OS credential store when it exists and falls back to the file.

The auth page describes no other setup step for the CLI. The desktop app and IDE extension have their own "Sign in another way" and "Use API Key" buttons.

## ChatGPT sign-in vs API key

| | ChatGPT sign-in | API key |
|---|---|---|
| Who pays | Your ChatGPT plan (included credits) | Your OpenAI Platform account, at standard API rates |
| Codex cloud | Yes | No, cloud requires ChatGPT sign-in |
| Local Codex CLI work | Yes | Yes |
| CI/CD jobs | Not named in the auth page | Named as the use case for programmatic workflows |
| Plugins | Supported OpenAI-curated plugins | Supported OpenAI-curated plugins, but some are not available |

Source: OpenAI's Codex auth page (learn.chatgpt.com/docs/auth, the current address of developers.openai.com/codex/auth), read 2026-11-19. Plan names and credit amounts change, so check the plan page before you decide.

## Lock Codex to one login type

A team can stop people from using the wrong login. The `forced_login_method` key in `config.toml` accepts `chatgpt` or `api`. The `forced_chatgpt_workspace_id` key limits ChatGPT logins to one workspace. Both are in the Codex config reference, read 2026-11-19. For the other keys, see [Codex CLI config reference](/articles/codex-cli-config-reference/).

To use your own model provider with a key, the config reference has `model_providers.<id>.env_key`, which names the environment variable that holds the provider key. The `experimental_bearer_token` key is marked "discouraged" there, so use `env_key`.

## Keep the key safe

- **Treat `~/.codex/auth.json` like a password.** OpenAI's page says Codex caches the login there in plain text, unless you pick a credential store.
- **Do not run Codex execution in untrusted or public places.** The page says this directly.
- **Use a separate key for CI.** OpenAI's page names CI/CD jobs as the use case for API keys. Our advice, not OpenAI's: give each job its own key so you can revoke one without the others.

## Memory and API keys

The API key only logs Codex in. It does not give Codex a memory. Codex keeps its own notes in `AGENTS.md` files, and a store that carries facts across sessions is a separate tool. [Codex memory](/articles/codex-memory/) covers the options.

One option is [Hindsight](https://github.com/vectorize-io/hindsight). Its Coding Agents package covers Codex CLI with `npx @vectorize-io/hindsight-coding-agents install codex`, which adds hooks and an MCP entry to your Codex config (it needs `codex_hooks = true`). It is one npm package for 22 CLI agents ([Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents), read 2026-11-12). Hindsight's benchmark scores on long histories are a reason to look at it: 64.1 on BEAM at 10M tokens, against 48.6 for Mem0's Platform. Both figures come from [Mem0's own comparison page](https://mem0.ai/compare/mem0-vs-hindsight), so they are vendor-run, not independent.

The limits matter. Hindsight's memory writes make LLM calls to extract facts, so a memory store adds cost on top of the Codex bill. The Hindsight server is MIT-licensed and can run on your own machine, but you have to set it up yourself. Its benchmark numbers do not show how a Codex API key bill compares with a memory bill; measure both on your own sessions.

Sources: [OpenAI Codex auth](https://learn.chatgpt.com/docs/auth) and [Codex config reference](https://learn.chatgpt.com/docs/config-file/config-reference) (both read 2026-11-19; the developers.openai.com links redirect there); [Hindsight README](https://github.com/vectorize-io/hindsight); [Hindsight coding agents docs](https://hindsight.vectorize.io/sdks/integrations/coding-agents) (read 2026-11-12); [Mem0 vs Hindsight](https://mem0.ai/compare/mem0-vs-hindsight).


## FAQ

### How do I log Codex CLI in with an API key?

Put the key in the OPENAI_API_KEY variable and run printenv OPENAI_API_KEY | codex login --with-api-key. OpenAI's Codex auth page, read 2026-11-19, gives this command. The key is then saved in ~/.codex/auth.json or in your OS credential store.

### Does Codex cost more with an API key?

Yes, it can. OpenAI's Codex auth page says that with an API key, Codex uses standard API pricing instead of the credits included in a ChatGPT plan. Usage is billed through your OpenAI Platform account.

### Can I use Codex cloud with an API key?

No. The Codex auth page says Codex cloud requires signing in with ChatGPT. An API key covers local Codex CLI work, including CI jobs.

