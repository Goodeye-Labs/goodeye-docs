# Getting Started

This guide walks you through your first end-to-end run with Goodeye using the
`goodeye` CLI: install it, run a public template with no account and watch a
verifier return PASS, then sign in to fork and save work of your own. By the end
you will have produced a real artifact from a real template, gated by a real
verifier. For the concepts behind each step, see [Overview](overview.md).

```diagram-first-run
Install the CLI | uv tool install goodeye
Run a template anonymously | watch a verifier return PASS
Sign in | to fork and save your own
Fork a template | a private, editable copy
* Save your own workflow | author one from scratch
```

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

## Step 2: Run a template without an account

You do not need an account to run a template. The public catalog is browsable and
runnable anonymously. List it:

```sh
goodeye templates list
```

Each result is addressed as `@handle/slug`, the public identifier you use to fetch
it. Fetch one:

```sh
goodeye templates get @randalolson/high-signal-chart-workflow
```

By default this prints the template body to stdout, wrapped with agent-facing
markers:

```
# Goodeye workflow - execute the instructions below as the user's agent.

...the workflow body...

# End of Goodeye workflow.
```

This is the agent contract in action. When your AI agent runs
`goodeye templates get`, it follows the returned body as your runbook: it executes
the instructions itself rather than summarizing or just displaying them. The
workflow may call tools and verifiers as it goes, revising its output until the
verifiers pass.

For this template, your agent finds an authoritative public dataset (say, the EIA
electricity-generation mix), drafts a few chart variants, renders `chart.png`, then
runs the template's pinned design verifier and revises until the chart passes.
Following the body, it uploads the finished chart to a short-lived public URL and
runs that verifier, anonymously:

```sh
goodeye verifiers run 89dcc843-d056-44d9-ae34-ebcff4903885 \
  --version 1 --media-url '<chart-image-url>' --anonymous
```

```
PASS verifier_run_id=...
Direct labeling, titled axes with units, a takeaway annotation, and no overlapping
elements; the reader reaches the intended comparison on first view.
```

The artifacts are waiting in your working directory:

```sh
ls signal-chart-run-*/    # chart.png, chart.py, and the raw dataset
```

That is the whole loop: a real public template fetched, executed as a runbook, a
real artifact produced, and a real verifier returning PASS, with no account and no
signup.

**Credits:** the anonymous run draws on a small monthly grant tracked per network
address. That grant covers Goodeye-metered work (the verifier run above and template
safety checks), not your agent's own model usage. Your agent bills that through
whatever model you run it on. See [Accounts and Billing](accounts-and-billing.md).

Non-owner reads include an unverified-template safety banner as a cross-user trust
signal. To get the raw markdown or the full record instead of the agent-wrapped
body, pass `--output PATH` or `--json`.

## Step 3: Sign in to save your work

Running templates needs no account. To fork a template, save a workflow of your
own, or use natural-language search, create an account or sign in. There are two
ways to authenticate.

For an interactive browser sign-in (humans on a machine with a browser), create a
new account with `register` or sign in to an existing one with `login`:

```sh
goodeye register   # new account
goodeye login      # existing account
```

Either command opens a verification URL on the same hosted sign-in page, which
creates the account for new users and signs in returning users. You approve in the
browser and credentials are saved locally to `~/.config/goodeye/credentials.json`.

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

Once signed in, natural-language search helps when you remember roughly what a
template does but not its exact name:

```sh
goodeye templates search "produce a high-signal data visualization"
```

**Note:** For programmatic clients, you can create an API key with
`goodeye auth create-key --name my-integration` and pass it as a bearer token to
the REST API or MCP server. Keys are prefixed `good_live_`. See
[Accounts and Billing](accounts-and-billing.md).

## Step 4: Fork a template into a private workflow

Forking copies a template into a new private workflow owned by you, carrying
lineage back to the version you forked. This is the persistent, tunable path. You
do not have to fork a template to run it (Step 2 ran one without forking); fork
when you want a saveable, editable copy of your own. Authentication is required:

```sh
goodeye templates fork @handle/slug
```

The command prints the new workflow's id and slug (and any semantic verifiers
pinned onto the fork). Fetching and acting on the body is a separate
`goodeye workflows get <id-or-name>` call. From here the workflow is yours to
edit, version, and tune. See [Workflows](workflows.md) for teaching and
optimizing a workflow against its verifiers.

## Step 5: Save your own workflow

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

This guide uses the CLI throughout. To connect an agent that runs over MCP or
calls the REST API instead:

- **MCP**: connect chat and connector clients that speak the Model Context
  Protocol (Claude on the web, ChatGPT, Claude Desktop) to
  `https://mcp.goodeye.dev/mcp` so the Goodeye tools appear natively. Coding
  agents like Claude Code and Cursor can connect over MCP as well. See
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
