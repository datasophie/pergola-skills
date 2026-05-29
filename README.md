# Pergola Skills

Agent skills for working with the Pergola deployment platform.

## Contents

- `pergola-mcp/` - operate Pergola via the `mcp__pergola__*` MCP tools. Preferred
  over `pergola-cli` whenever the MCP tools are available in the session
- `pergola-cli/` - operate the `pergola` command-line tool for tasks that
  genuinely require the shell binary (access keys, CLI profiles, private
  repo creds, interactive `local-connect`).
- `pergola-manifest/` - author, validate, and debug Pergola project manifests
  such as `pergola.yaml`.

Each skill includes a `SKILL.md` file and any supporting reference material under
its `references/` directory.

## Usage

Install or copy the skill directories into your Agents skills directory, then ask
your Agent for help with Pergola tasks. For example:

```text
Use the pergola-cli skill to deploy this project to my dev stage.
```

```text
Use the pergola-manifest skill to create a pergola.yaml for this app.
```

See the individual skill files for detailed workflows and command references.

## Requirements

The CLI-focused skill assumes the Pergola CLI is available locally:

```sh
curl -fsSL https://get.pergo.la/cli/latest/install.sh | bash
```

For other installation options and platforms (i.e. Windows), see [get.pergo.la/cli](https://get.pergo.la/cli).
