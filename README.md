# opsvault-skill

[Agent Skill](https://agentskills.io) that teaches AI coding agents (Codex, Claude Code, Cursor, etc.) how to drive the [OpsVault](https://github.com/chenxi2015/opsvault) CLI to install, manage, and operate CentOS/Linux middleware and web services.

## What's inside

```
opsvault/
├── SKILL.md              # Agent instructions (frontmatter + playbook)
└── agents/
    └── openai.yaml       # UI metadata for Codex
```

`SKILL.md` covers:

- Golden rules (CLI over TUI, sudo, confirm-before-purge, Nginx binary-only, ...)
- First-time install of the `opsvault` binary itself (curl one-liner, China mirror, version pin)
- Per-service lifecycle cheatsheet (MySQL / Redis / Nginx / RocketMQ / RabbitMQ / Kafka / Postgres / MinIO / Nacos / ELK / GitLab / Jenkins / Prometheus / Grafana / Node Exporter / Qdrant / Milvus / Ollama)
- Nginx vhost + SSL workflows
- Ansible fleet ops (`ping` / `exec` / `doctor` / `deploy` / `push` / `uninstall`)
- Kubernetes subcommands
- Troubleshooting and "when NOT to use" boundaries

## Install the skill into your agent

Pick one of the standard skill roots; the agent scans them at startup.

```bash
# Codex (per-user)
git clone https://github.com/<you>/opsvault-skill.git ~/opsvault-skill
ln -s ~/opsvault-skill/opsvault ~/.codex/skills/opsvault

# Or cross-agent shared root
ln -s ~/opsvault-skill/opsvault ~/.agents/skills/opsvault
```

Prefer a symlink so the skill tracks repo updates. A plain `cp -r` also works.

## Verify

Restart your agent, then ask: **"用 opsvault 部署一个 mysql"** — the agent should load this skill and follow the golden rules (CLI over TUI, no `--mode docker` for nginx, confirm before `--purge`).

## Authoring conventions

- Frontmatter `name` must equal the directory name (`opsvault`).
- `description` is the trigger text the agent matches against; keep it keyword-rich and include an explicit "when NOT to use" boundary.
- Keep the body under ~200 lines; link out to the [OpsVault README](https://github.com/chenxi2015/opsvault) for deep architecture detail instead of duplicating it here.

## License

Same as the upstream OpsVault project.
