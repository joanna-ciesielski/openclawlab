# OpenClaw Audit Lab

A self-directed lab: deploy [OpenClaw](https://openclaw.ai) (a multi-agent gateway) in Docker, build a small agent fleet, and perform a security-and-cost audit of the running system against a professional 12-point scope. Built to develop and demonstrate AI-infrastructure audit capability on a system I own end to end.

**Honest framing:** this is my own lab instance, not client or production work. Every finding was tested or measured on the running deployment, not asserted from documentation.

---

## What this project demonstrates

- **Containerized AI-agent infrastructure** — a working OpenClaw deployment on Docker via the official install path, plus a from-scratch adapted image for a network-restricted environment.
- **Multi-agent orchestration** — a four-agent fleet spanning hosted and local models, an external tool server over MCP, scheduled automation, hooks, and session state.
- **Security & cost auditing** — a structured audit against a 12-point scope producing 12 findings (2 High) and 3 checked-clean items, two remediated with before/after evidence.
- **Model routing judgment** — deliberate hosted-vs-local decisions, measured rather than assumed.

## What was built

**Deployment.** OpenClaw running under Docker Compose using the official image (`ghcr.io/openclaw/openclaw:latest`, v2026.7.1) on Apple Silicon, with agent, workspace, and credential state redirected to auditable host paths and secrets handled through the project's supported reference mechanism (no keys committed; `.env` at mode 600). A full deployment map documents where agents, skills, hooks, cron, session/memory state, credentials, and logs reside on disk (host and container), which ports are exposed, and what runs as which user.

**Adapted image (restricted-network dry run).** The install was first validated in an egress-restricted sandbox where all container registries were blocked. This required building a base image from the host root filesystem (`docker import`), sourcing a compliant Node.js runtime from PyPI, and installing OpenClaw from npm — then running it under the *unmodified* official Compose file. Every deviation is documented for audit fidelity (`Dockerfile.sandbox`, `SETUP_NOTES.md`).

**Four-agent fleet.**
- `main` and `finance` on Claude Sonnet 5 (hosted) — orchestration and tool-driven reasoning.
- `summarizer` and `digest` on qwen2.5:3b via Ollama (local, on-device) — cheap, repetitive work.
- A custom Python MCP server (read-only finance tooling) bridged from stdio to Streamable HTTP with a non-invasive launcher, so the containerized gateway could reach a host process via `host.docker.internal` — including diagnosing and working around the MCP SDK's DNS-rebinding host-header protection. The finance agent then called the live tools and returned real query results.
- A scheduled cron job, two hooks (session-memory and a command audit-logger), and session/memory state across all agents.

## The audit (the product)

Method: inspect the running deployment and read its own outputs (`docker inspect`/`port`, the `openclaw` CLI, on-disk config and state, cron run history for token counts), plus live behavioral tests — verifying each finding against the system before recording it.

Representative findings:

- **Network exposure (High, remediated).** The gateway published on `0.0.0.0` (and IPv6) while its config recorded a loopback bind — the gateway logged the mismatch itself. Remediated via a Compose override restricting ports to `127.0.0.1` and removing an unused published port; re-verified.
- **Prompt-injection posture (High).** An embedded hostile instruction was resisted by the hosted model but only *incidentally* blocked on the local model (context overflow, not defense); a scheduled agent separately attempted a write outside its workspace, stopped only by non-root container permissions. Injection defense rested on the model alone, with no sandbox and over-broad default tools.
- **Cost / context (Medium, root-caused).** A two-sentence scheduled digest consumed a measured **16,400 input tokens** because full bootstrap context loaded on every call — traced to the fix of moving fixed-format output to code and trimming context.
- **Reliability & logging.** Runtime logs written to a non-persistent container path; a scheduled job failing closed with no delivery route, silently dropping output.
- **Checked-clean (rigor).** Container hardening (non-root uid 1000, dropped capabilities, `no-new-privileges`, no host Docker socket mounted), secrets-at-rest, and a persisted config-change audit trail.

Full write-up: the two-page audit report and the long-form findings document.

## Repository contents

| File | What it is |
|---|---|
| `SETUP_NOTES.md` | Running log of every command and decision, including friction and surprises |
| `DEPLOYMENT_MAP.md` | Where every component (agents, skills, hooks, cron, state, credentials, logs) resides on disk; ports; process users |
| `Dockerfile.sandbox` | The adapted image for the restricted-network dry run, with each deviation documented |
| `OpenClaw_Audit_Report.pdf` | Two-page audit report (finding → evidence → severity → impact → fix) |
| `.gitignore` | Excludes secrets and runtime state from version control |

## Tech & skills

Docker · Docker Compose · Linux containers · Model Context Protocol (MCP) · local LLMs (Ollama) · Claude API · Python · security auditing (network exposure, credential handling, prompt injection, least-privilege) · cost/token optimization · model routing · git.

## Key outcomes

- Deployed and audited a Dockerized multi-agent AI gateway (OpenClaw) against a 12-point security-and-cost scope; produced a documented deployment map and a two-page findings report with 12 findings (2 High) and remediations verified by re-test.
- Built a four-agent fleet mixing hosted (Claude) and on-device local (Ollama) models, integrating a custom Python MCP tool server bridged from stdio to HTTP across the host/container boundary.
- Identified and remediated a network-exposure misconfiguration (gateway bound to all interfaces despite a loopback config) via a Docker Compose override, verified with `docker port`.
- Measured a 16,400-token cost anomaly on a trivial scheduled task and root-caused it to unbounded context loading; specified a code-over-model fix.
- Tested prompt-injection resistance across hosted and local models and documented that container hardening (non-root, dropped capabilities) — not policy — was the effective control, recommending sandboxing and least-privilege tooling.

---

_Self-audit of a personal lab deployment, performed against a professional audit scope. No client or production data involved._
