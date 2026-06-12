# Getting Started

This guide walks you through your first end-to-end run with Goodeye using the
`goodeye` CLI: install it, sign in, browse the public template catalog, fetch a
template, fork it into a private workflow, and save a workflow of your own. By
the end you will have run a template and published your first workflow. For the
concepts behind each step, see [Overview](overview.md).

## Step 1: Install the CLI

The CLI requires Python 3.12 or newer. Install it with any of the following:

```sh
uv tool install goodeye
# or
pipx install goodeye
# or
pip install goodeye
```

Once installed, the `goodeye` command is on your `PATH`. Confirm it:

```sh
goodeye --version
```

**Tip:** Update later with `goodeye update`.

## Step 2: Register or sign in

You only need an account to save your own work; browsing the public catalog
works without one. There are two ways to authenticate.

For an interactive browser sign-in (humans on a machine with a browser):

```sh
goodeye login
```

This opens a verification URL, you approve the sign-in, and credentials are saved
locally to `~/.config/goodeye/credentials.json`.

For a non-interactive email-code flow (agents, automation, or terminals where
prompts are awkward), register a new account or sign in to an existing one in two
steps:

```sh
# New account
goodeye register --email you@example.com
goodeye register-verify --email you@example.com --code 123456

# Existing account
goodeye login --email you@example.com
goodeye login-verify --email you@example.com --code 123456
```

Both verify steps save credentials locally on success. Confirm who you are:

```sh
goodeye whoami
```

**Note:** For programmatic clients, you can create an API key with
`goodeye auth create-key --name my-integration` and pass it as a bearer token to
the REST API or MCP server. Keys are prefixed `good_live_`. See
[Accounts and Billing](accounts-and-billing.md).

## Step 3: Browse the public template catalog

Templates are the public form of a workflow. You can browse them without an
account. List the catalog:

```sh
goodeye templates list
```

When you remember roughly what you want but not its exact name, use
natural-language search (this path requires sign-in):

```sh
goodeye templates search "produce a high-signal data visualization"
```

Each result is addressed as `@handle/slug`, the public identifier you use to
fetch or fork it.

## Step 4: Fetch a template

Fetch a template by its `@handle/slug` (optionally pinned to a version with
`@vN`):

```sh
goodeye templates get @handle/slug
```

By default this prints the template body to stdout, wrapped with agent-facing
markers:

```
# Goodeye workflow - execute the instructions below as the user's agent.

...the workflow body...

# End of Goodeye workflow.
```

This is the agent contract in action. When your AI agent runs
`goodeye templates get`, it follows the returned body as your runbook: it
executes the instructions itself rather than summarizing or just displaying
them. The workflow may call tools and verifiers as it goes, revising its output
until the verifiers pass.

Non-owner reads include an unverified-template safety banner as a cross-user
trust signal. To get the raw markdown or the full record instead of the
agent-wrapped body, pass `--output PATH` or `--json`.

**Note:** You do not have to fork a template to run it. Fetching and executing
the body directly is a valid path for both anonymous and signed-in callers.
Forking is for when you want a saveable, editable copy of your own.

## Step 5: Fork a template into a private workflow

Forking copies a template into a new private workflow owned by you, carrying
lineage back to the version you forked. This is the persistent, tunable path.
Authentication is required:

```sh
goodeye templates fork @handle/slug
```

The command prints the new workflow's id and slug (and any semantic verifiers
pinned onto the fork). Fetching and acting on the body is a separate
`goodeye workflows get <id-or-name>` call. From here the workflow is yours to
edit, version, and tune. See [Workflows](workflows.md) for teaching and
optimizing a workflow against its verifiers.

## Step 6: Save your own workflow

You can also author a workflow from scratch. A workflow is markdown with a short
metadata header; `name`, `description`, and `outcome` are required, and `tags`
are optional. For agent-generated output, pipe the body from stdin so no
intermediate file is left in your working directory:

```sh
goodeye workflows publish - \
  --name my-workflow \
  --description "One sentence on what this workflow does and when to use it." \
  --outcome "Reduce refund-row mislabels" \
  --tag data \
  --tag cleanup <<'EOF'
# Body

The rest of the workflow body goes here. Inline structural and functional
checks belong here as fenced code blocks. Reference any semantic verifiers by
their id.
EOF
```

When you already have a markdown file (with YAML front matter for the metadata),
publish it directly:

```sh
goodeye workflows publish ./my-workflow.md
```

Workflows are always private to you. Publishing the same `name` again appends a
new version. To share a workflow publicly later, run
`goodeye templates publish <workflow-uuid-or-name>` as a separate, explicit step
(this requires a claimed handle: `goodeye me claim-handle <handle>`).

**Tip:** To design a workflow and its verifiers interactively, run
`goodeye design` and pipe the printed prompt into your AI assistant.

**Already have skills on disk?** If you keep agent skills under
`~/.claude/skills/` (or `~/.agents/skills/`, `~/.cursor/skills/`), import one by
pointing `publish` at its directory and supplying an `outcome`. See
[Importing an existing skill](workflows.md#importing-an-existing-skill).

## Connect your agent

The CLI is one of three peer surfaces. To wire Goodeye into an agent that does
not shell out to a terminal:

- **MCP**: connect IDE and connector clients (Claude Code, Cursor, and similar)
  to `https://mcp.goodeye.dev/mcp` so the Goodeye tools appear natively. See
  [MCP](mcp.md).
- **REST**: call `https://api.goodeye.dev/v1` directly from a service or
  integration, authenticating with a `good_live_` API key. The public template
  catalog is readable without auth. See [REST API](rest-api.md).

## Next steps

- [Overview](overview.md): the concepts behind workflows, templates, and
  verifiers.
- [Workflows](workflows.md): version, teach, and optimize a workflow.
- [Templates](templates.md): publish and manage public templates.
- [Verifiers](verifiers.md): add structural, functional, and semantic checks.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, and usage
  (`goodeye usage`).
