# Docker Sandbox Cheatsheet

Quick reference for the `sbx` CLI (Docker Sandboxes). Verified against the docs for **sbx v0.48.0** (Oct 2026).
Docs: https://docs.docker.com/ai/sandboxes/ · CLI ref: https://docs.docker.com/reference/cli/sbx/

## What it is

- Runs AI coding agents in an isolated **microVM**. Each sandbox has its own private Docker daemon.
- All outbound traffic goes through a **host-side proxy** that allows or blocks it against your network policy.
- Credentials stay on the host. The proxy injects them into requests, so the agent never sees the raw token.
- You don't need Docker Desktop. The CLI and local sandboxes are free, including for commercial use.
- Supported agents: `claude`, `codex`, `copilot`, `cursor`, `devin`, `docker-agent`, `droid`, `gemini`, `kiro`, `opencode`, `shell` (a plain shell with no agent)

## Install & sign in

| OS         | Install                        | Upgrade                     |
| ---------- | ------------------------------ | --------------------------- |
| Windows 11 | `winget install -h Docker.sbx` | `winget upgrade Docker.sbx` |
- **Windows:** enable the hypervisor platform once, from an admin PowerShell: `Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All`

```bash
sbx login                     # browser OAuth to Docker
sbx policy init balanced      # first-time network policy: allow-all | balanced | deny-all
sbx secret set github --command 'gh auth token'   # let the agent use GitHub
```

## Everyday commands

```bash
sbx                           # interactive dashboard (same as `sbx tui`)
sbx run claude                # create + attach, mounts the current dir
sbx run --name my-sb claude   # same, with an explicit name
sbx run --name my-sb          # reattach from anywhere (agent read from the sandbox)
sbx ls                        # list sandboxes (status, ports, workspace)
sbx stop my-sb                # stop it; state is kept
sbx rm my-sb                  # delete it and everything inside (host files untouched)
sbx prune                     # remove all *stopped* sandboxes
sbx run --rm claude           # throwaway: removed when the agent exits
sbx run -d --name my-sb claude .   # start in background, no session
```

Pass arguments to the agent after `--`:

```bash
sbx run claude -- --continue                      # flags are added to the defaults
sbx run --name my-sb claude -- "Fix the login bug"  # start with a prompt
```

> Claude starts with `claude --dangerously-skip-permissions` by default. If the first argument after `--` is a plain word instead of a flag (like `agents`), it **replaces** the defaults.

## Workspaces & mounting

| Command | Current dir mounted? |
|---|---|
| `sbx run claude` | **Yes** |
| `sbx run claude .` | **Yes** |
| `sbx run claude C:\Dev\Project` | Yes, the path you give |
| `sbx create --name test claude` | **No** (mountless) |
| `sbx create --name test claude .` | **Yes** |

- `sbx create` makes a sandbox without attaching. Attach later with `sbx run --name <name>`.
- The first path is the **primary** workspace. It's mounted at the same absolute path inside the sandbox.
- Add `:ro` to make an extra path read-only:

```powershell
sbx create claude --name my-sb `
  "C:\Dev\Playground" `
  "C:\Dev\Shared:ro"
```

## Git modes

| | Direct (default) | Clone (`--clone`) |
|---|---|---|
| Agent edits | Your host working tree | A private clone inside the sandbox |
| Changes visible on host | Immediately | After `git fetch sandbox-<name>` or after the agent pushes |
| Best for | Working turn-by-turn on one branch | Parallel agents, keeping your tree clean |

```bash
sbx run --clone claude .                 # clone mode (can only be set at creation)
git fetch sandbox-<name>                 # on the host: pull in the agent's branches
git diff main..sandbox-<name>/<branch>
```

- Clone mode follows whatever branch the host had checked out at creation. It doesn't create a branch for you.
- The `sandbox-<name>` remote only works while the sandbox is running.
- `sbx rm` deletes the clone, so **fetch or push your work first**.

## Network policy

```bash
sbx policy ls                                  # active rules
sbx policy log [my-sb]                         # what was allowed/blocked (start here when something fails)
sbx policy allow network registry.npmjs.org    # global allow
sbx policy allow network "*.npmjs.org,*.pypi.org"
sbx policy allow network --sandbox my-sb api.example.com   # one sandbox only
sbx policy deny network evil.example.com
sbx policy rm network --resource api.example.com
sbx policy allow network "**"                  # allow everything (use sparingly)
sbx policy reset                               # start over
```

- If `allow` doesn't unblock a request, your **org policy** may be overriding it. Ask IT.

## Ports & host services

```bash
sbx run -p 8080:3000 --name my-sb claude   # publish at creation (host 8080 → sandbox 3000)
sbx ports my-sb --publish 8080:3000        # publish on an existing sandbox
sbx ports my-sb --publish 3000             # random host port; run `sbx ports my-sb` to see it
sbx ports my-sb --unpublish 8080:3000
```

- Services inside the sandbox must listen on `0.0.0.0` (or `[::]`), not just `127.0.0.1`.
- To reach a service on your **host** from inside the sandbox:
  1. Use `host.docker.internal:<port>`, not `localhost`.
  2. Allow it in the policy: `sbx policy allow network localhost:<port>`.

## Secrets & environment variables

```bash
sbx secret set anthropic                   # interactive prompt (services: anthropic, github, openai, ...)
sbx secret set openai --sandbox my-sb      # scoped to one sandbox
sbx secret set github --command 'gh auth token'   # resolve from a command on the host
sbx secret ls
sbx run -e LOG_LEVEL=debug --env-file .env.sandbox claude
```

- Use `sbx secret` for keys and tokens. The proxy injects them, so the agent never sees the raw value.
- Plain `-e` env vars **are** visible to anything running in the sandbox.
- To keep an env var across sessions on an existing sandbox:
  `sbx exec my-sb bash -c "echo 'export FOO=bar' >> /etc/sandbox-persistent.sh"`

## Working inside a sandbox

```bash
sbx exec -it my-sb bash                    # shell into the sandbox
sbx exec -u root my-sb apt-get update      # run as root
sbx cp ./config.json my-sb:/home/agent/workspace/   # sandbox paths must be absolute
sbx cp my-sb:/home/agent/workspace/out.log ./
```

- Your host's `~/.claude` (user settings, hooks) is **not** brought into the sandbox. Project-level config in the repo is.
- Skills are the exception: `sbx skills import` or `sbx skills add` puts them in a shared store all sandboxes can use.
- Updating `sbx` doesn't update the agent inside existing sandboxes. Run the agent's own update command inside the sandbox.

## Shared agent skills

- Install skills from **Git repositories** (`sbx skills add`) or import them from supported agents on your **host** (`sbx skills import`) to use in local sandboxes.
- They live in a persistent store that **survives sandbox deletion**.
- By default, the store is shared with every new sandbox that runs a supported agent.

## Kits & templates (customizing)

[V3 kits documentation](https://github.com/docker/sandbox-kit-spec?utm_source=chatgpt.com)

- **Kits** package tools, config, network rules and credentials for a sandbox (Early Access). Built-in agent names use **v2** kits:
  - `kind: mixin`: adds capabilities to an existing agent. You can stack several.
  - `kind: sandbox`: defines a full agent (image, entrypoint, policies).
  - v3 kits call these *workload* and *mixin*. Every kit in a sandbox must be v3 or every kit v2; you can't mix them.

```powershell
sbx create claude --name my-sb --kit C:\Dev\sandbox-kits .   # add a mixin at creation
sbx kit add my-sb ./my-mixin/                                # add to an existing sandbox (experimental)
```

- **Templates** snapshot a configured sandbox so you can reuse it:

```bash
sbx template save my-sb my-template:v1                  # add --output file.tar to share it
sbx run --pull never -t my-template:v1 claude
sbx template ls / sbx template rm my-template:v1 / sbx template load file.tar
```

> Templates include any secrets you put in files by hand. Use `sbx secret` so they stay out.

- **`sbxenv.yaml`** (experimental) is like a Compose file for a sandbox: agent, workspace, kits, env, ports. Use `sbx env run` / `plan` / `create` / `rm`. Keep the file **outside** the mounted workspace. [Docs](https://docs.docker.com/ai/sandboxes/configuration/environment-files/)

## Settings

```bash
sbx settings ls
sbx settings get clipboard.imagePaste
sbx settings set clipboard.imagePaste true    # let Ctrl+V paste images (off by default)
sbx settings set claude.remoteControl true    # enable Claude's /remote-control
sbx settings unset <key>
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| Anything weird | `sbx diagnose` (`--upload` to send the report to Docker) |
| Commands hang or show daemon errors | `sbx daemon restart` (sandbox data is kept) |
| `sbx ls` shows *not responding* | `sbx stop <name>`, then `sbx run --name <name>` |
| Can't install packages or reach an API | `sbx policy log`, then `sbx policy allow network <host>` |
| Out of disk (root is 20 GB by default) | Recreate with `DOCKER_SANDBOXES_ROOT_SIZE=40g sbx run claude` |
| Can't reach a host service | Use `host.docker.internal` and allow `localhost:<port>` |
| Nuclear option | `sbx reset` wipes **all** sandboxes, policies and secrets (`--preserve-secrets` keeps secrets) |

Check your version with `sbx version`.
