# OpenClaw Docker Deployment Map

**Verified against a live deployment** — OpenClaw 2026.7.1-2 (build 0790d9f), Docker Engine 29.4.3, Compose v5.1.3, 2026-08-14.
Every path below was observed on the running system unless marked *(doc-derived)*.

**Environment caveat:** this instance runs in a cloud Linux sandbox, image `openclaw:sandbox`
(adapted; see `Dockerfile.sandbox` for the four documented deviations). The *topology* —
compose file, mounts, ports, users, caps — is the stock upstream `docker-compose.yml` at
commit `18163ba269c3`. On the Mac, host paths become `/Users/joanna/...` and the image
becomes `ghcr.io/openclaw/openclaw:latest`; everything in the container column is unchanged.

## Topology

Compose project `openclaw`, rooted at `~/Projects/openclaw-lab/openclaw/` (the upstream clone — the compose file, `.env`, and any `docker-compose.extra.yml` all live here; **this directory is the deployment root an auditor should ask for**).

| Service | Lifecycle | Purpose |
|---|---|---|
| `openclaw-gateway` | long-running, `restart: unless-stopped` | the product: HTTP gateway + agent runtime |
| `openclaw-cli` | ephemeral `run` containers | CLI; shares the gateway's network namespace (`network_mode: service:openclaw-gateway`) — CLI↔gateway traffic never leaves the container pair |

## Who runs as what

| Where | Identity | Evidence |
|---|---|---|
| Container PID 1 | `/sbin/docker-init` (tini equivalent, from compose `init: true`), user `node` | `ps aux` in container |
| Gateway process | `node dist/index.js gateway --bind lan --port 18789`, user `node` = **uid 1000, gid 1000, non-root** | `ps aux`, `id` |
| Host side of bind mounts | uid 1000 = `ubuntu` on this host (would be `joanna` uid 501 via Docker Desktop's uid mapping on macOS) | `ls -la state/` |
| ⚠ Sandbox-image quirk | the renamed `node` user inherited supplementary groups incl. `sudo` from the imported rootfs — **not present in the official image**; flag if seen in a real deployment | `id` output |

Security posture (stock compose): `cap_drop: NET_RAW, NET_ADMIN`; `no-new-privileges:true`; no docker.sock mount (sandbox mode off; the compose file's commented-out docker.sock mount is the switch to look for in audits — mounting it grants effective host root).

## Ports

| Host binding | Container | What it is |
|---|---|---|
| `0.0.0.0:18789` | 18789 | Gateway Control UI + API. Health: `/healthz` (unauthenticated liveness), `/startupz`, `/readyz`. Auth: bearer `OPENCLAW_GATEWAY_TOKEN` |
| `0.0.0.0:18790` | 18790 | device-pair / file-transfer bridge |
| `0.0.0.0:3978` | 3978 | MS Teams webhook port (open even with no Teams channel configured) |

⚠ **Audit finding, reproducible on any host:** compose publishes on `0.0.0.0` and launches the
gateway with `--bind lan`, which **overrides** the `gateway.bind: "loopback"` written to
`openclaw.json` by onboarding. The gateway itself logs
`⚠ Gateway is binding to a non-loopback address`. Loopback-only intent requires editing the
compose port mappings (e.g. `127.0.0.1:18789:18789`) — the config file alone does not deliver it.

## Filesystem map (host ⇄ container)

Three bind mounts (host locations chosen via `OPENCLAW_CONFIG_DIR` / `OPENCLAW_WORKSPACE_DIR` /
`OPENCLAW_AUTH_PROFILE_SECRET_DIR` at setup time; defaults would be `~/.openclaw` and
`~/.openclaw-auth-profile-secrets`):

| Host (this lab) | Container | Holds |
|---|---|---|
| `~/Projects/openclaw-lab/state/openclaw` | `/home/node/.openclaw` | everything below |
| `~/Projects/openclaw-lab/state/openclaw/workspace` | `/home/node/.openclaw/workspace` | agent working dir |
| `~/Projects/openclaw-lab/state/openclaw-auth-profile-secrets` | `/home/node/.config/openclaw` | OAuth-profile encryption key (empty here — API-key auth in env-ref mode never creates it) |

Inside `…/.openclaw` (container paths; prepend the host mount to translate):

| Path | Mode | Contents |
|---|---|---|
| `openclaw.json` (+ `.bak`, `.bak.N`, `.last-good`) | 600 | behavior config; gateway token stored **as env reference only** |
| `agents/main/` | | the one agent ("main") |
| `agents/main/agent/models.json` | 600 | per-agent model prefs |
| `agents/main/agent/plugins/` | 700 | per-agent plugin state |
| `agents/main/agent/openclaw-agent.sqlite` | — | per-agent auth store (named by `models status`; created lazily — not yet on disk in this deployment) |
| `agents/main/sessions/<uuid>.jsonl` | 600 | **full message transcripts**, one per session |
| `agents/main/sessions/<uuid>.trajectory.jsonl` | 600 | tool/step trajectory |
| `agents/main/sessions/sessions.json` | 600 | session index |
| `agents/main/sessions/skills-prompts/` | | rendered skill prompts |
| `state/openclaw.sqlite` (+ `-wal`, `-shm`) | 600 | gateway state DB (cron jobs, queues, etc. live here once created) |
| `identity/device.json` | 600 | device identity/keys |
| `logs/config-audit.jsonl` | 600 | **config-change audit trail** (host-persisted — useful) |
| `plugin-skills/<name>/` | 755 | skills shipped by plugins (`browser-automation`, `canvas` present) |
| `skill-workshop/proposals(.json)` | 755 | agent-authored skill drafts |
| `workspace/` | 700 | agent workspace — OpenClaw git-inits it; `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `TOOLS.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` are the agent's memory/persona files (session-memory hook writes here) |
| `workspace-attestations/<sha256>.attested` | 755 | workspace attestation marker |

## Agents, skills, hooks, cron

**Agents.** One agent `main` (default), model `anthropic/claude-sonnet-5`, routing default
(no rules). Everything about it lives under `agents/main/` above. More agents ⇒ more
`agents/<id>/` siblings.

**Skills.** Plugin-shipped skills under `plugin-skills/`; agent-drafted skills under
`skill-workshop/`; per-session rendered prompts under `agents/main/sessions/skills-prompts/`.
Skill installs use npm (config `skills.install.nodeManager: "npm"`).

**Hooks.** Config-driven: `hooks.internal.entries.session-memory: enabled` in `openclaw.json`
is the only hook active. No user hook scripts on disk. (The `.sample` files in
`workspace/.git/hooks/` are inert git defaults, not OpenClaw hooks.)

**Cron.** `openclaw cron list` → "No cron jobs." Definitions are created via the CLI/config
and persist in the config/state store (`openclaw.json` / `state/openclaw.sqlite`)
*(doc-derived — none exist to observe)*. There is no OS crontab in the container.

## Credentials — every place a secret touches disk

| # | Location (host) | What | Form | Notes |
|---|---|---|---|---|
| 1 | `openclaw/.env` — root:root **600** | `ANTHROPIC_API_KEY`, `OPENCLAW_GATEWAY_TOKEN` | **plaintext** | THE secret file. Compose `env_file` injects it into both services. Git-ignored upstream and in this lab repo. Created by `setup.sh` (token) — note it existed *before the install ever succeeded* |
| 2 | `state/openclaw/openclaw.json` | gateway token | env **reference** (`{"source":"env","id":"OPENCLAW_GATEWAY_TOKEN"}`) | no secret material; result of `--secret-input-mode ref` |
| 3 | gateway process environment | both secrets | plaintext in env | visible via `docker inspect` (Config.Env) and `/proc/7/environ` to anyone with docker access — standard Docker caveat |
| 4 | `state/openclaw/identity/device.json` | device identity | keys | 600, uid 1000 |
| 5 | `state/openclaw-auth-profile-secrets/` | OAuth-profile encryption key | — | empty in API-key deployments; populated only for OAuth auth |
| 6 | `agents/main/agent/openclaw-agent.sqlite` | per-agent auth store | — | lazy-created; would hold profile records (refs in ref-mode) |

Anthropic key verification: `models status` shows
`anthropic … source=env: ANTHROPIC_API_KEY` — resolved from environment at runtime,
never copied into state. Grep of `state/` for the key found nothing.

## Logs

| Log | Where | Persisted? |
|---|---|---|
| Gateway runtime log | container `/tmp/openclaw-1000/openclaw-YYYY-MM-DD.log` | ❌ **lost on container recreation** — /tmp is container-local, not mounted. Audit implication: default deployment keeps no durable runtime log |
| stdout/stderr | Docker json-file driver, host `/var/lib/docker/containers/<id>/…-json.log` | survives restarts, lost on `rm`; readable via `docker logs` |
| Config audit trail | host `state/openclaw/logs/config-audit.jsonl` | ✅ durable — records every config mutation |
| Session transcripts | host `state/openclaw/agents/main/sessions/*.jsonl` | ✅ durable — full conversation content at rest, mode 600 |

## Quick audit checklist derived from this map

1. Who can read `openclaw/.env`? (all secrets, plaintext)
2. Are ports 18789/18790/3978 bound to 0.0.0.0? (default: yes, despite config saying loopback)
3. Is docker.sock mounted? (off by default; sandbox mode wants it — trust boundary)
4. Are session transcripts (`sessions/*.jsonl`) in backups / synced dirs?
5. Is the runtime log expectation documented? (container-local by default)
6. Container user non-root uid 1000, caps dropped, no-new-privileges — confirm not overridden
7. `config-audit.jsonl` — use it; it's the change history
