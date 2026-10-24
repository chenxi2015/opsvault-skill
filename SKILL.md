---
name: opsvault
description: Drive the OpsVault CLI to install, manage, and operate CentOS/Linux middleware and web services. Use when the user wants to install/start/stop/upgrade/uninstall MySQL, Redis, Nginx, RocketMQ, RabbitMQ, Kafka, PostgreSQL, MinIO, Nacos, ELK, GitLab, Jenkins, Prometheus, Grafana, Node Exporter, Qdrant, Milvus, or Ollama; manage Nginx vhosts and SSL certificates; batch-operate clusters via Ansible; or inspect Kubernetes nodes/pods/services through opsvault. Do NOT use for generic docker/kubectl/ansible commands unrelated to opsvault.
metadata:
  short-description: Drive the OpsVault ops CLI for middleware and cluster management
---

# OpsVault

OpsVault (`opsvault`) is a one-stop CentOS middleware and web-service ops CLI/TUI. Docker is the default deploy driver; Nginx is binary-only (source-compiled by OpsVault itself); Ansible and Kubernetes have dedicated subcommand groups.

## Golden rules

1. **Prefer CLI subcommands over TUI.** `opsvault tui` is interactive and blocks the terminal; only use it when the user explicitly asks for the TUI.
2. **Root/sudo is required** for systemd, ports 80/443, and the docker daemon. If a command fails with permission denied, retry with `sudo` and say so.
3. **Never hand-edit files OpsVault owns**: anything under `/data/opsvault/`, `/usr/local/nginx/`, the nginx systemd unit, or logrotate configs. Always mutate through opsvault subcommands so the driver layer stays the source of truth.
4. **Nginx is binary-only.** Never pass `--mode docker` to `opsvault nginx`, never suggest `docker run nginx` as a substitute.
5. **Confirm before destructive actions**: any `uninstall` (especially `--purge`), `opsvault ansible uninstall --purge`, `opsvault nginx vhost del --delete-root`, `opsvault k8s delete`. State what will be removed and wait for a yes.
6. **Default mode is docker.** Only pass `--mode binary` or `--mode k8s` when the user asks or the subcommand requires it.

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

- `<service>` ∈ `mysql redis rocketmq rabbitmq kafka postgres minio nacos elk gitlab jenkins prometheus grafana nodeexporter qdrant milvus ollama`
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

RocketMQ extras:

```bash
opsvault rocketmq version      # broker version
opsvault rocketmq dlq stat     # dead-letter queue stats
```

### Nginx (binary only)

```bash
opsvault nginx install            # source-compile nginx+pcre+openssl, register systemd
opsvault nginx start|stop|restart|reload|status|log
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

### Kubernetes

```bash
opsvault k8s get pods|nodes|services
opsvault k8s status
opsvault k8s doctor
opsvault k8s log <pod-name>
opsvault k8s dashboard
opsvault k8s install|delete|uninstall
opsvault k8s register
```

### Top-level helpers

- `opsvault doctor` — host environment health check
- `opsvault init` — first-run config wizard (interactive)
- `opsvault bak` — backup flows
- `opsvault migrate` — host/engine migration
- `opsvault upgrade` — self-upgrade the opsvault binary

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
