# Pergola CLI — full command reference

Run `pergola <verb> <noun> --help` for exact, current flags. Resource-scoped
commands take `-p/--project` (optional with a default project) and, where
noted, `-s/--stage`.

## Contents
- Standalone commands
- create / push / add / remove
- list
- logs
- Lifecycle / state
- update / delete / restore
- enable / disable
- CLI config
- notifications / validate

## Standalone
- `pergola login --endpoint <uri>` — OIDC device-flow login.
- `pergola exec <component> -p -s -- <cmd...>` — run a command in a component.
- `pergola local-connect <component> [[host:]localPort:]remotePort -p -s` — port-forward (alias `port-forward`).
- `pergola info` — CLI version / build / auth info (aliases `version`, `about`).
- `pergola mcp serve` — run Pergola as an MCP server over stdio (see "MCP server setup" in operations.md).
- `pergola rest` — low-level REST passthrough (hidden/experimental).

## create
- `create project <project> --git-url <url> [--display-name <name>]`
- `create stage <stage> -p --type <dev|qa|prod> [--display-name <name>] [--outpost-uri <uri>]`
- `create access-key`
- `create backup -p -s --display-name <name>`
- `create pat -p --name <name> --token <secret>` — store/replace a project's git token (PAT) for HTTPS clone URLs (alias `personal-access-token`).
- `create ssh -p` — generate a project's git SSH key pair; returns public key + fingerprint (alias `ssh-key`).

## push
- `push build -p [--branch <branch>] [--commit <sha>] [--force]` (`--commit` is a 7-40 lowercase hex SHA on the selected `--branch`)
- `push release -p -s [-b <build>] [-c <config>] [--when-ready]` (at least one of `-b`/`-c`; `--when-ready` needs `-b`, waits up to 30m for that build to succeed, and pushes nothing if it fails)

## add
- `add config-data <config> -p -s [--env KEY=VALUE ...] [--file <path> ...]`
- `add member <member> -p [--role member|owner]`
- `add auto-build -p [--renew-secret]`
- `add auto-release -p -s --branch <branch> [-c <config>] [--disabled]`
- `add config-identity <config> -p -s --identity <identity> [--aws-role-arn <arn> | --azure-client-id <id> | --gcp-service-account <acct>]`

## remove
- `remove config-data <config> -p -s --key <k> [--key <k> ...]`
- `remove member <member> -p`
- `remove auto-build -p`
- `remove auto-release -p -s --branch <branch> [-c <config>]`
- `remove config-identity <config> -p -s --identity <identity>`

## list (alias plural forms, e.g. `projects`)
- `list project [--all | --archived]`
- `list stage -p [--archived]`
- `list build -p`
- `list release -p [-s]`
- `list component -p -s`
- `list config -p -s`
- `list config-data <config> -p -s [--with-values]`
- `list member -p`
- `list access-key`, `list cli-config`
- `list backup -p -s`, `list auto-build -p`, `list auto-release -p -s`
- `list config-identity <config> -p -s`
- `list commits <build> -p`
- `list vulnerabilities <build> -p [-c <component>] [--id <vulnerability-id>]`
- `list pat -p` — show the project's git PAT name (value never shown), if configured (alias `personal-access-token`).
- `list ssh -p` — show the project's git SSH public key + fingerprint, if configured (alias `ssh-key`).

## logs
- `logs build <build> -p [--query] [--since | --since-time] [--follow]`
- `logs component <component> -p -s [--query] [--since | --since-time] [--follow]`
- `logs stage <stage> -p [--query] [--since | --since-time] [--follow]`

## Lifecycle / state
- `start component <component> -p -s [--now]`
- `stop component <component> -p -s [--kill]`
- `restart component <component> -p -s`
- `suspend stage <stage> -p`
- `resume stage <stage> -p`

## update / delete / restore
- `update project ...`, `update stage <stage> -p [--type] [--display-name]`
- `delete project <project>`, `delete stage <stage> -p`, `delete access-key <key-id>`, `delete config <config> -p -s`, `delete cli-config <name>`
- `delete pat -p` (alias `personal-access-token`), `delete ssh -p` (alias `ssh-key`) — remove a project's git PAT / SSH key pair
- `restore project <project>`, `restore stage <stage> -p`, `restore backup <backup-id> -p -s`

## enable / disable
- `enable access-key`, `disable access-key`

## CLI config
- `set cli-config [--config <name>] [--default-project <name>] [--endpoint <uri>]`
- `use cli-config <name>`

## notifications / validate
- `notifications project <project> [--since | --since-time]`
- `validate manifest <path>` — validate a `pergola.yaml`/`pergola.yml`/`pergola.json`.
