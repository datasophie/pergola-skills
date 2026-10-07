# Pergola Skills

Agent skills for working with the Pergola deployment platform.

## Contents

- `pergola-mcp/` - operate Pergola via the `mcp__pergola__*` MCP tools. Preferred
  over `pergola-cli` whenever the MCP tools are available in the session
- `pergola-cli/` - the full manual for the `pergola` command-line tool. Used
  when the MCP tools are not available, and for CLI-only tasks (login, access
  keys, CLI profiles, private repo creds, interactive `local-connect`).
- `pergola-manifest/` - author, validate, and debug Pergola project manifests
  such as `pergola.yaml`.

Each skill includes a `SKILL.md` file and any supporting reference material under
its `references/` directory.

## Usage

Install or copy the skill directories into your Agents skills directory, then ask
your Agent for help with Pergola tasks. For example:

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

## Requirements

The CLI-focused skill assumes the Pergola CLI is available locally:

```sh
curl -fsSL https://get.pergo.la/cli/latest/install.sh | bash
```

For other installation options and platforms (i.e. Windows), see [get.pergo.la/cli](https://get.pergo.la/cli).

## Skill guidelines

This is what every skill in this repository has to look like.

### Layout

```text
pergola-example/
├── SKILL.md         required, the only file an agent loads by itself
├── references/      optional, detail an agent reads when a task needs it
│   └── topic.md
└── scripts/         optional, helpers an agent runs instead of reading
    └── wait-for-x.sh
```

- The directory name is the skill's `name`.
- Nothing else belongs in a skill directory, so no README, notes or test data.
  Everything in it ships to users.

### Frontmatter

- **`name`** uses lowercase letters, digits and hyphens, at most 64 characters.
  It uses the `pergola-` prefix like the existing skills.
- **`description`** has at most 1024 characters and about 150 tokens
  because it is loaded in every session. Write it in the third
  person ("Operates…", "Authors…"). Say what the skill does and when to use
  it, and name every kind of action that changes live systems, such as
  deploys, deletions or access changes. If another skill covers similar
  ground, say which one to prefer.
- Only `name` and `description` are used in this repository.

### SKILL.md

Keep the body under 500 lines and the whole file under about 2,500 tokens.
Every SKILL.md has these parts, in this order:

1. **Title and intro** One short paragraph on what the skill covers and what
   it assumes.
2. **The common path** The steps most tasks need, as a numbered list or one
   command block.
3. **Verification**, usually titled "Done means verified". For each step, say
   what success looks like, what failure looks like, and what to do next. Name
   exact output strings and fields instead of "check that it worked", and say
   when an exit code can't be trusted.
4. **Confirmation**, only for skills that can change live systems. List the
   operations by kind (delete or remove, restore, suspend, access and
   credential changes), and say that a request naming the operation and its
   target already counts as a yes.
5. **Gotchas** One bullet per lesson: a short bold title, what happens, and
   what to do instead.
6. **More** One line per reference file saying when to read it. Then a
   sentence telling the agent that when a tool contradicts the skill, it
   trusts the tool and tells the user which statement is outdated.

Skill-specific sections, such as a mental model, conventions or a decision
guide, can go anywhere between the intro and the gotchas.

### References and scripts

- Link every reference file directly from SKILL.md, so no file is reachable
  only through another reference.
- Name each reference after what it holds, such as `recipes.md` or
  `operations.md`. Above 100 lines, it starts with a contents list.
- Write a script when a procedure is deterministic and the agent would
  otherwise improvise it, like polling or parsing output. First check whether
  a tool or CLI flag already does it, such as `pergola_wait_for_release` or
  `pergola push release --when-ready`.
- Scripts are run, not read. SKILL.md shows the exact command and what comes
  back. A script exits non-zero on failure and prints what to fix.

### Writing

- **Only what the agent can't know** Don't repeat tool descriptions, `--help`
  output, error messages or general knowledge. Point to them instead.
- **Exact names** Copy tool names, flags, fields and error strings from the
  source code or the official docs, and check them there.
- **Verified facts only** Every rule and gotcha is confirmed in code, docs or
  a real run. If something is only suspected, leave it out. When behavior
  depends on a version, name it.
- **Timeless wording** No dates and no words like "currently" or "recently".
  Describe how things work, and name versions where behavior differs.
- **Consistent terms** Use the product's word for each thing, and only that
  word.
- **Short and direct** Write instructions as imperatives. Put commands in
  code blocks with placeholders such as `my-project`, `dev` or `<build>`, and
  put success criteria and comparisons in tables.

### Safety

- Never reveal secret values in the context. Use placeholders such as `<secret>`, tell the
  agent not to echo values back, and keep access keys in the client
  configuration, never in the chat.
- Link to installers instead of piping a download into a shell.
- Never tell the agent to skip confirmations or to edit skill files. Lessons
  reach a skill through a person, prompted by the sentence in "More".
- A script may install packages, use the network or read credentials only if
  the SKILL.md line that calls it says so.
