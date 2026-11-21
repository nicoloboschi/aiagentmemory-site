# OpenClaw Docker Setup (2026): Run the Gateway in a Container

> How to run the OpenClaw Gateway in Docker with the setup script, a pre-built image or the browser image, what to set in .env, and where memory fits. Steps from the OpenClaw Docker docs, read 2026-11-21.

Source: https://aiagentmemory.org/articles/openclaw-docker/ · Updated 2026-11-21 · aiagentmemory.org


## How do you run OpenClaw in Docker?

Run `./scripts/docker/setup.sh` from the root of the OpenClaw repository. The script builds the Gateway image as `openclaw:local`, asks for your model provider API keys, writes a Gateway token to `.env`, and starts the Gateway with Docker Compose. Then open `http://127.0.0.1:18789/` and paste the token into Settings. The steps come from the [OpenClaw Docker docs](https://docs.openclaw.ai/install/docker), read 2026-11-21.

Docker is optional. The docs suggest it for a throwaway Gateway or a machine where you do not want local installs. If you develop on your own computer, the normal install is simpler.

## What do you need before you start?

The docs list these prerequisites:

- Docker Desktop, or Docker Engine, with Docker Compose v2.
- At least 6 GB of RAM for a local source build. A pre-built image skips this build.
- Enough disk space for images and logs.
- On a public server, the security hardening steps for network exposure. The docs name the Docker `DOCKER-USER` firewall chain.

## What are the setup options?

| Option | How to set it | When to use it |
|---|---|---|
| Local source build (default) | `./scripts/docker/setup.sh` | You have the repo and 6 GB of RAM to spare. |
| Pre-built image | `export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"` before setup | You want to skip the build. The docs say to use the GHCR image or the Docker Hub mirror `openclaw/openclaw`, not unofficial mirrors. |
| Pinned release | A version tag such as `2026.9.3` in `OPENCLAW_IMAGE` | You want the same version on every machine. |
| Browser image | `OPENCLAW_IMAGE` set to a `-browser` tag, such as `ghcr.io/openclaw/openclaw:latest-browser` | You want the Gateway to control a Chromium browser inside the container. |
| Offline host | `docker load -i openclaw-image.tar`, then `./scripts/docker/setup.sh --offline` | The host has no internet. The image must already be loaded. |

The docs say the `latest` and `main` tags move with each stable release. Pin a version tag if your setup must not change on its own.

## What does the setup script do?

The script runs in four steps:

1. It builds the image, or pulls the one you named.
2. It runs onboarding: it asks for provider API keys, generates a Gateway token and writes it to `.env`.
3. It starts the Gateway container with Docker Compose.
4. It fixes file permissions and syncs the Gateway config.

Onboarding runs through the `openclaw-gateway` container, not the CLI container. The CLI container shares the Gateway's network, so it only works after the Gateway exists.

## How do you open the dashboard again?

If you forget the URL, run this command:

```bash
docker compose run --rm openclaw-cli dashboard --no-open
```

If you changed `OPENCLAW_GATEWAY_PORT`, replace port 18789 in the printed URL with your own host port. Keep the rest of the URL as it is.

## How do you add a chat channel?

The docs show the commands for three channels, each run through the CLI container:

```bash
docker compose run --rm openclaw-cli channels login
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
```

The first command pairs WhatsApp with a QR code. The other two take a bot token.

## How does the browser work in Docker?

The Control UI has a browser panel. That browser runs in the Gateway, not on your laptop. For a local browser inside a Docker Gateway, Chromium must exist inside the container. The `-browser` image tags ship Chromium already. For a new install, set `OPENCLAW_IMAGE` to `ghcr.io/openclaw/openclaw:latest-browser` before you run setup.

The docs warn about one case. A home volume or bind mount that covers `/home/node/.cache/ms-playwright` can hide the Chromium in the image. If the browser is not found after you switch images, check those mounts first.

## Where does memory fit?

OpenClaw's memory plugin for Hindsight is `@vectorize-io/hindsight-openclaw`. Its v1.2.0 release, from 2026-11-03, adds `hindsight_retain`, `hindsight_recall` and `hindsight_reflect` tools and a `memoryMode` option (`hybrid`, `context` or `tools`). The release notes are on Hindsight's GitHub.

The Docker guide does not mention memory. We did not run the plugin inside the container, so we cannot say it works there. Check the plugin's docs and test it on a copy of your data first. For the plugin itself, read [OpenClaw AI agent memory](/articles/openclaw-ai-agent-memory/).

Hindsight is open source under the MIT license and can run on your own machine. Its server starts with one `docker run` command and keeps its data in a volume. Hindsight's own benchmark leads are on long histories: 64.1 on BEAM 10M against 48.6 for Mem0. Both figures come from Mem0's comparison page, so they are not independent. Hindsight also returns more context per recall than Mem0 by default, and each retain calls an LLM, which costs tokens.

## Frequently asked questions

**How do I run OpenClaw in Docker?** Run `./scripts/docker/setup.sh` from the repo root, then open `http://127.0.0.1:18789/` and paste the token from `.env` into Settings.

**Does the OpenClaw Docker setup include memory?** The Docker guide does not describe a memory plugin. The Hindsight plugin, `@vectorize-io/hindsight-openclaw`, was not tested inside the container for this page.

**Which image should I use?** Use the local build if you have 6 GB of RAM to spare. Use the pre-built `latest` image to skip the build, and a `-browser` tag if you need Chromium in the container.


## FAQ

### How do I run OpenClaw in Docker?

Clone the OpenClaw repository, run ./scripts/docker/setup.sh from its root, and open http://127.0.0.1:18789/ once the Gateway starts. The script builds the image locally, asks for your provider API keys, writes a Gateway token to .env, and starts the Gateway with Docker Compose. Docker is optional; the OpenClaw docs say to use the normal install if you develop on your own machine.

### Does the OpenClaw Docker setup include memory?

The Docker guide does not describe a memory plugin. OpenClaw's Hindsight plugin is the package @vectorize-io/hindsight-openclaw. Its Docker setup was not tested for this page, so check the plugin's own docs before you rely on it inside the container.

### Which OpenClaw Docker image should I use?

Use the default setup for a local build, which needs about 6 GB of RAM. Set OPENCLAW_IMAGE to ghcr.io/openclaw/openclaw:latest to skip the build. Use the latest-browser tag if you want the Gateway-controlled browser, which needs Chromium inside the container.

