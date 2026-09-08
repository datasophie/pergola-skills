# Pergola Skills

Agent skills for working with the Pergola cloud deployment platform.

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

Install or copy the skill directories into your Agents skills directory.

The easiest way to install and manage skills is via:
```shell
npx skills add datasophie/pergola-skills
```

You can also manually install/copy the skill directories into your Agent, e.g. into `~/.agents/skills/`.
Please refer to your Agent's documentation for further details and specifics.

Once installed, you can ask your Agent for help with Pergola tasks, for example:

```text
Use the pergola-mcp init tool to pergolize this project
```

```text
Use the pergola-mcp skill to deploy this project to my dev stage.
```

```text
Use the pergola-cli skill to set up an access key for CI.
```

```text
Use the pergola-manifest skill to create a pergola.yaml for this app.
```

See the individual skill files for detailed workflows and command references.

Further documentation is also available [here](https://docs.pergola.cloud/docs/tutorials/agentic-devops).

## Requirements

The `pergola-cli` skill assumes the Pergola CLI is available locally.

You can download it from: [get.pergo.la/cli](https://get.pergo.la/cli)
