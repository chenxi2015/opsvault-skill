---
name: opsvault
description: Drive the OpsVault CLI to install, manage, and operate CentOS/Linux middleware and web services. Use when the user wants to install/start/stop/upgrade/uninstall MySQL, Redis, Nginx, RocketMQ, RabbitMQ, Kafka, PostgreSQL, MinIO, Nacos, ELK, GitLab, Jenkins, Prometheus, Grafana, Node Exporter, Qdrant, Milvus, or Ollama; manage Nginx vhosts and SSL certificates; batch-operate clusters via Ansible; inspect Kubernetes nodes/pods/services through opsvault; back up or restore service configs; or migrate services between hosts/engines. Do NOT use for generic docker/kubectl/ansible commands unrelated to opsvault.
metadata:
  short-description: Drive the OpsVault ops CLI for middleware and cluster management
---

# OpsVault

OpsVault (`opsvault`) is a one-stop CentOS middleware and web-service ops CLI/TUI. Docker is the default deploy driver; Nginx is binary-only (source-compiled by OpsVault itself); Ansible and Kubernetes have dedicated subcommand groups.

## Golden rules

1. **二次确认铁律 — always get explicit user confirmation before any sensitive action, even when the tool would let it through.** "Sensitive" includes: any `uninstall` (with or without `--purge`), any `--purge` flag, `opsvault ansible exec` classified as mutating or dangerous, `opsvault ansible uninstall`, `opsvault nginx vhost del`, `opsvault nginx ssl delete`, `opsvault k8s delete`, `opsvault migrate host/engine`, `opsvault bak restore/delete`, any service `stop`/`restart` on a production-looking group, and any command targeting `--group all` or `-e prod`. **How to confirm:** state the exact command, the blast radius (which hosts / which data), and what will be lost if it goes wrong — then wait for an explicit "yes" / "确认" / "go". Do not interpret silence, "ok", "嗯", or a thumbs-up as consent. If the user is not in the loop (CI, automation), do NOT run the sensitive action — surface it and stop.
2. **Prefer CLI subcommands over TUI.** `opsvault tui` is interactive and blocks the terminal; only use it when the user explicitly asks for the TUI.
3. **Root/sudo is required** for systemd, ports 80/443, and the docker daemon. If a command fails with permission denied, retry with `sudo` and say so.
4. **Never hand-edit files OpsVault owns**: anything under `/data/opsvault/`, `/usr/local/nginx/`, the nginx systemd unit, or logrotate configs. Always mutate through opsvault subcommands so the driver layer stays the source of truth.
5. **Nginx is binary-only.** Never pass `--mode docker` to `opsvault nginx`, never suggest `docker run nginx` as a substitute.
6. **Default mode is docker.** `--mode` only accepts `docker|binary`. Only pass `--mode binary` when the user asks or the subcommand requires it (Nginx is the only binary-only service; everything else defaults to docker).
7. **When in doubt, ask opsvault itself.** Every command and subcommand has built-in help: `opsvault <cmd> --help` and `opsvault <cmd> <subcmd> --help`. If a flag or subcommand isn't listed in this skill, run `--help` on the live binary before guessing — the CLI is the source of truth, this file is a map.

## First-time install

If `opsvault` is not on the target host (`command -v opsvault` fails), install it before doing anything else.

```bash
# Global
curl -fsSL https://raw.githubusercontent.com/chenxi2015/opsvault/main/scripts/install.sh | bash

# China mirror (recommended on CN networks)
curl -fsSL https://ghfast.top/https://raw.githubusercontent.com/chenxi2015/opsvault/main/scripts/install.sh | bash

# Pin a version or override install dir
curl -fsSL https://ghfast.top/https://raw.githubusercontent.com/chenxi2015/opsvault/main/scripts/install.sh \
  | bash -s -- --version v0.1.0 --install-dir /data/opsvault
```

Install layout:

- **Linux (root)** → `/data/opsvault` (binary at `/data/opsvault/bin/opsvault`, config at `/data/opsvault/configs/default.yaml`, symlinked to `/usr/local/bin/opsvault`)
- **macOS / non-root** → `~/.opsvault` (config at `~/.opsvault/configs/default.yaml`)

After install, verify with `opsvault version`, then proceed to Preflight. For upgrades on a host that already has opsvault, prefer `opsvault upgrade` over re-running the install script.

## Preflight

```bash
opsvault --version
opsvault doctor
```

`opsvault doctor` checks docker, systemd, ports, and config health. Run it on a fresh host or when the user reports a broken environment.

Config search order (first match wins): `--config <path>` → `./configs/default.yaml` → exe-adjacent `configs/` → `~/.opsvault/configs/default.yaml` → `/data/opsvault/configs/default.yaml` → `/etc/opsvault/default.yaml`. To see which file is active, run any command with `--debug` and read the viper load log. First run on a fresh host: `opsvault init` (interactive wizard) or write a yaml and pass `--config`.

## Command reference

### Per-service lifecycle

Pattern: `opsvault <service> <action>`.

- `<service>` ∈ `mysql redis rocketmq rabbitmq kafka postgres minio nacos elk gitlab jenkins prometheus grafana node-exporter qdrant milvus ollama` — note **`node-exporter`** uses a hyphen, not a run-together word
- `<action>` ∈ `install start stop restart status log upgrade uninstall`

Common flags:

- `--mode docker|binary|k8s` — driver selection (default `docker`; only some services support binary/k8s)
- `--purge` — on `uninstall`, also delete the host data dir under `/data/opsvault/<service>/`
- `--tag <image-tag>` — on `upgrade`, target image version
- `--bind-ip 0.0.0.0|127.0.0.1` — docker port bind address
- `--debug` — verbose logs

Service-specific install flags:

```bash
opsvault mysql install [--root-pwd xxx]
opsvault redis install [--pwd xxx]
opsvault rabbitmq install [--admin-user admin --admin-pwd xxx]
opsvault minio install [--pwd xxx] [--random-pwd]   # --random-pwd generates + persists a strong password
```

When credential flags are omitted and a password is required, OpsVault auto-generates a strong password and prints it once — capture and relay it to the user.

Interactive shells (run in the foreground, drop user into a REPL/CLI):

```bash
opsvault mysql cli        # mysql client inside the container
opsvault mysql exec ...   # run a one-off mysql command
opsvault redis cli        # redis-cli inside the container
opsvault rocketmq sh      # mqadmin shell
```

RocketMQ extras:

```bash
opsvault rocketmq version      # broker version
opsvault rocketmq dlq stat     # dead-letter queue stats
```

### Nginx (binary only)

```bash
opsvault nginx install            # source-compile nginx+pcre+openssl, register systemd
opsvault nginx start|stop|restart|reload|status|log
opsvault nginx log rotate         # force logrotate run
opsvault nginx module             # manage compiled-in modules
opsvault nginx upgrade
opsvault nginx uninstall [--purge]

opsvault nginx vhost add --domain example.com --root /data/wwwroot/example
opsvault nginx vhost add --domain api.example.com --proxy 8080
opsvault nginx vhost del --domain example.com [--delete-root]
opsvault nginx vhost list

opsvault nginx ssl apply  --domain example.com
opsvault nginx ssl renew  --domain example.com     # or renew all
opsvault nginx ssl delete --domain example.com
opsvault nginx ssl cron enable|disable|status      # auto-renew cron
```

Fixed paths (from config): program `/usr/local/nginx`, webroot `/data/wwwroot`, SSL `/data/ssl`, logs `/data/wwwlogs`.

### Ansible cluster ops

Requires `configs/ansible.yaml` inventory (template: `configs/ansible.example.yaml` in the opsvault repo). All commands take `[-e test|prod]` to pick the environment.

```bash
opsvault ansible ping    --group db_servers [-e test]
opsvault ansible exec    --group db_servers --cmd "uptime"
opsvault ansible doctor  --group db_servers
opsvault ansible list
opsvault ansible deploy    --service mysql --group db_servers
opsvault ansible uninstall --service mysql --group db_servers [--purge]
opsvault ansible push      --group db_servers --bin ./bin/opsvault-linux-amd64 --config-path ./configs/default.yaml
```

`ansible deploy` auto-generates a 20-char strong password when credentials aren't set — surface the highlighted card to the user verbatim.

### Ansible exec — remote shell safety

`opsvault ansible exec --group <g> --cmd "<shell>"` fans a shell string out to every host in the group via ansible's `shell` module. OpsVault ships a built-in safety gate that classifies every command into **safe / mutating / dangerous** before anything is sent:

- **safe** (read-only) — runs immediately. No confirmation required by the tool.
- **mutating** (service restart, file write, package install, redirect, ...) — interactive confirmation required unless `--yes` is passed.
- **dangerous** (`rm -rf`, `mkfs`, `dd of=/dev/...`, `shutdown`, `reboot`, `iptables -F`, `useradd`/`userdel`/`passwd`, `crontab -e/-r`, `yum remove`/`update -y`, `curl ... | bash`, base64-obfuscated payloads, fork bombs, ...) — **refused by default**; requires `--allow-destructive` AND interactive confirmation. Even when the user asks, you must apply Golden rule #1 (二次确认铁律) on top.

Every invocation — including blocked and dry-run attempts — is appended to the opsvault audit log with the full command, group, risk level, and matched reason.

**Useful flags:**

```bash
--dry-run              # print parsed command + risk level, do not ssh
--limit <host>         # ansible -l, run on one host first
--timeout <sec>        # wrap remote command in `timeout <sec> bash -c ...`
--yes, -y              # skip interactive confirm (still blocked for dangerous)
--allow-destructive    # permit dangerous commands (still needs confirm / --yes)
```

**Standard flow when the user asks you to run something mutating or unfamiliar across a group:**

```bash
# 1. dry-run to see classification
opsvault ansible exec --group db --cmd "<cmd>" --dry-run

# 2. probe one host first
opsvault ansible exec --group db --limit <one-host> --cmd "<cmd>"

# 3. only then widen
opsvault ansible exec --group db --cmd "<cmd>"
```

**Hard rules for this skill:**

1. **Never run `exec` against `--group all` or a production group on the first try.** Always `--limit` to a single representative host first, then widen.
2. **Read-only by default.** If the user only wants information, do NOT pipe into mutating commands (`xargs rm`, `tee`, `>` redirects to system paths, etc.).
3. **Never bypass the safety gate.** Do not pre-emptively add `--allow-destructive` or `--yes` to "save a step". Pass them only after the user has explicitly confirmed under Golden rule #1.
4. **Banned patterns — even with `--allow-destructive`, refuse unless the user has explicitly described that exact change AND you have done 二次确认.** This list is not exhaustive; use judgment.
   - File destruction: `rm -rf`, `rm -fr`, `rm -r /`, `find ... -delete`, `shred`, `wipefs`, `mkfs.*`, `dd of=/dev/...`
   - Privilege/identity: `useradd`, `userdel`, `passwd`, `chpasswd`, edits to `/etc/sudoers`, `visudo`, `chmod 777`, `chown -R` on `/` or `/etc`
   - Service/disruption: `shutdown`, `reboot`, `halt`, `poweroff`, `init 0|6`, `systemctl stop docker|sshd|network*`, `kill -9 -1`, `pkill -9 -u`
   - Network/fork bombs: `:(){ :|:& };:`, `while true; do ... done` without a clear exit, `iptables -F`, `nft flush ruleset`
   - Persistence/backdoors: writing to `crontab`, `/etc/rc.local`, `~/.ssh/authorized_keys`, systemd unit drop-ins, unless the user explicitly described that exact change
   - Mass package ops: `yum remove`, `yum update -y`, `apt remove`, `dnf autoremove` (use `opsvault ansible deploy/uninstall` instead — it's idempotent and audited)
   - Obfuscation that hides intent: base64-decoded payloads, `${IFS}` tricks, hex/unicode-escaped commands, `eval "$(curl ...)"`. If you can't read the command aloud safely, don't send it.
5. **Mutating but routine ops** (restart a service, rotate a log, clean a specific app's tmp dir) — these are fine after 二次确认, BUT:
   - Prefer the dedicated opsvault subcommand if one exists (`opsvault ansible deploy`, `opsvault ansible uninstall`, `opsvault <svc> restart` via local ssh) — those are idempotent, logged, and roll back cleaner than ad-hoc shell.
   - Quote all paths, avoid globbing outside the intended directory, and add a guard like `test -d /data/opsvault/mysql && ...` when touching app data.
6. **Capture output.** If output is long or the user will want it later, redirect to a file under the local opsvault log dir and surface the path.

**Patterns that ARE safe to run without asking twice:**

```bash
opsvault ansible exec --group db -c "uptime"
opsvault ansible exec --group db -c "df -h / && free -m"
opsvault ansible exec --group web -c "systemctl is-active nginx || true"
opsvault ansible exec --group db  -c "docker ps --format 'table {{.Names}}\t{{.Status}}'"
opsvault ansible exec --group db  -c "tail -n 50 /data/opsvault/logs/opsvault.log"
```

**Patterns that REQUIRE explicit user confirmation + narrowed scope:**

```bash
# restarting a service across a group
opsvault ansible exec --group web -c "systemctl restart nginx"

# clearing a specific, known tmp dir (note the guard)
opsvault ansible exec --group app -c "test -d /data/opsvault/app/tmp && find /data/opsvault/app/tmp -mindepth 1 -maxdepth 1 -mtime +7 -delete"

# disk pressure investigation that MIGHT lead to cleanup — investigate first, propose cleanup separately
opsvault ansible exec --group db -c "du -sh /data/opsvault/* 2>/dev/null | sort -hr | head"
```

**Refusal template** — if the user asks for something on the banned list, say no concretely and offer the safer path:

> "我不会用 `ansible exec` 跑 `rm -rf /data/opsvault/mysql` —— 这会绕过 opsvault 的驱动层直接删数据，audit 只能记录命令本身，救不回来。安全路径是 `opsvault ansible uninstall --service mysql --group db --purge`，它会先停容器、再卸载、最后清数据，每一步都可追溯。要不要走这条？"

### Kubernetes

```bash
opsvault k8s get pods|nodes|services
opsvault k8s status
opsvault k8s doctor
opsvault k8s log <pod-name>
opsvault k8s dashboard
opsvault k8s pod [pod-name]       # inspect a single pod
opsvault k8s install              # deploy opsvault-managed k8s agent
opsvault k8s uninstall            # remove the agent
opsvault k8s register             # register external service into k8s dns
opsvault k8s delete               # delete a k8s resource (confirm first)
```

`k8s delete` removes arbitrary k8s resources — treat it as destructive and always confirm with the user before running.

### Config backup & restore

OpsVault keeps timestamped snapshots of every service's config under its data root. Use these before upgrades or migrations.

```bash
opsvault bak create [service]           # snapshot one service (or all if omitted)
opsvault bak list                       # show existing snapshots
opsvault bak restore <backup_name> [service]
opsvault bak delete <backup_name>
```

### Host/engine migration

```bash
opsvault migrate host           # move service data+configs to another host group
opsvault migrate engine         # convert running service between binary and docker modes
opsvault migrate status         # show in-flight migration tasks
```

`migrate engine` is the supported path for switching a service from binary to docker (or vice versa) without data loss — do NOT manually uninstall+reinstall to change modes.

### Top-level helpers

- `opsvault doctor` — host environment health check
- `opsvault init` — first-run config wizard (interactive)
- `opsvault bak` — config backup/restore (see above)
- `opsvault migrate` — host/engine migration (see above)
- `opsvault upgrade` — self-upgrade the opsvault binary
- `opsvault version` — print version + platform info
- `opsvault completion [bash|zsh|fish|powershell]` — generate shell completion

## Workflow recipes

Deploy MySQL on a fresh host:

```bash
opsvault doctor
opsvault mysql install        # auto-creates opsvault-net bridge, maps 3306
opsvault mysql status
```

Stand up an HTTPS site behind Nginx:

```bash
opsvault nginx install                                    # one-time, source compile
opsvault nginx vhost add --domain app.example.com --proxy 8080
opsvault nginx ssl apply --domain app.example.com         # Let's Encrypt + force HTTPS
opsvault nginx ssl cron enable
```

Rolling upgrade a Docker middleware:

```bash
opsvault <svc> status                 # record current tag + ports
opsvault <svc> upgrade --tag <new>    # stops old container, pulls new image, reuses data dir
opsvault <svc> log                    # verify clean startup
```

Batch fleet health check:

```bash
opsvault ansible ping   --group all -e prod
opsvault ansible doctor --group all -e prod
```

## Troubleshooting

- **Config not found** → `opsvault init` or pass `--config`.
- **Docker daemon unreachable** → `opsvault doctor` reports it; on CentOS the tool can auto-install docker-ce, or the user runs `systemctl start docker`.
- **Port already in use** → `opsvault doctor` lists occupied ports; stop the conflicting service via opsvault rather than `kill -9`.
- **Permission denied** → re-run with `sudo`.
- **See what it's doing** → append `--debug`; audit log lives at `<log.storage_path>/opsvault.log` (default `/data/opsvault/logs/opsvault.log`).

## When NOT to use this skill

- Editing files inside opsvault-managed directories directly — use the CLI.
- One-off docker/kubectl experiments unrelated to the supported middleware list — call the underlying tool directly.
- Anything that needs a GUI or long-lived interactive session — opsvault is a batch CLI.
