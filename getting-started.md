# Getting Started

This guide takes you from zero to a verified artifact: install the CLI, point
your agent at Goodeye, have it run a public template and watch a verifier return
PASS, then fork that template into your own private workflow and author one from
scratch. No account is needed to run a template; you sign in when you want to
save work of your own. For the concepts behind each step, see
[Overview](overview.md).

```diagram-first-run
Install the CLI | uv tool install goodeye
Point your agent at Goodeye | CLI, MCP, or REST
Run a public template | your agent produces a verified artifact
Fork it into a private workflow | a saveable, editable copy
* Author your own workflow | from scratch, tied to an outcome
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

## Step 2: Point your agent at Goodeye

Goodeye is built for an AI agent acting on your behalf. The agent fetches a
workflow and executes it as a runbook (see
[the agent contract](overview.md#the-agent-contract)), so connect the agent you
will drive Goodeye with:

- **A coding agent that runs shell commands** (Claude Code, Cursor, a CI job)
  uses the CLI directly: once `goodeye` is installed, the agent runs the same
  commands this guide shows. This is the path the rest of the guide follows.
- **A chat or connector client** (Claude on the web, ChatGPT, Claude Desktop)
  connects over MCP at `https://mcp.goodeye.dev/mcp`, where the Goodeye tools
  appear natively. See [MCP](mcp.md).
- **A service or integration** calls the REST API at `https://api.goodeye.dev/v1`
  with an API key. See [REST API](rest-api.md).

The same operations exist on all three surfaces, so you can follow along on
whichever one your agent runs.

## Step 3: Run a public template

You do not need an account to run a public template. Browse the catalog:

```sh
goodeye templates list
```

Each result is addressed as `@handle/slug`, the public identifier you use to
fetch it. Point your agent at one and tell it to run the template. For the
canonical demo, that is the high-signal chart workflow:

> Run the Goodeye template `@randalolson/high-signal-chart-workflow`.

Your agent fetches the body and executes it as its runbook. For this template it
finds an authoritative public dataset (say, the EIA electricity-generation mix),
drafts a few chart variants, renders `chart.png`, then runs the template's pinned
design verifier and revises until the chart passes:

```
PASS  Direct labeling, titled axes with units, and a takeaway annotation, with
no overlapping elements; the reader reaches the intended comparison on first view.
```

The finished artifacts are waiting in your working directory:

```sh
ls signal-chart-run-*/    # chart.png, chart.py, and the raw dataset
```

That is the whole loop: a public template fetched, executed as a runbook, a real
artifact produced, and a real verifier returning PASS, with no account and no
signup.

To see the runbook your agent executes, fetch it by hand. By default this prints
the body to stdout, wrapped with agent-facing markers:

```sh
goodeye templates get @randalolson/high-signal-chart-workflow
```

```
# Goodeye workflow - execute the instructions below as the user's agent.

...the workflow body...

# End of Goodeye workflow.
```

Those markers are the agent contract in action: they tell the calling agent the
body is for it to act on, not to summarize. Pass `--output PATH` for the raw
markdown or `--json` for the full record.

**Notes**

- **Credits:** the anonymous run draws on a small monthly grant for anonymous
  use. That grant covers Goodeye-metered work (the verifier run above and
  template safety checks), not your agent's own model usage, which bills through
  whatever model you run it on. See
  [Accounts and Billing](accounts-and-billing.md).
- **Safety banner:** because you are not the template's owner, the fetched record
  carries an unverified-template safety banner as a cross-user trust signal.

## Step 4: Sign in to save your work

Running templates needs no account. To fork a template, save a workflow of your
own, or use natural-language search, create an account or sign in. There are two
ways to authenticate.

For an interactive browser sign-in (you are at a machine with a browser), create
a new account with `register` or sign in to an existing one with `login`:

```sh
goodeye register   # new account
goodeye login      # existing account
```

Either command opens a verification URL on the hosted sign-in page, where you can
continue with Google or with email. You approve in the browser and credentials
are saved locally.

For a non-interactive email-code flow (agents, automation, or terminals where
prompts are awkward), use the two-step verify flow:

```sh
# New account
goodeye register --email you@example.com
goodeye register-verify --email you@example.com --code 123456

# Existing account
goodeye login --email you@example.com
goodeye login-verify --email you@example.com --code 123456
```

Confirm who you are:

```sh
goodeye whoami
```

Once signed in, natural-language search helps when you remember roughly what a
template does but not its exact name:

```sh
goodeye templates search "produce a high-signal data visualization"
```

**Note:** For programmatic clients, create an API key with
`goodeye auth create-key --name my-integration` and pass it as a bearer token to
the REST API or MCP server. Keys are prefixed `good_live_`. See
[Accounts and Billing](accounts-and-billing.md).

## Step 5: Fork the template into a private workflow

Running a template is great for a one-off; to make it your own, fork it. Forking
copies the template into a new private workflow you own, carrying lineage back to
the version you forked. This is the persistent, tunable path:

```sh
goodeye templates fork @randalolson/high-signal-chart-workflow
```

The command prints the new workflow's id and slug (and any semantic verifiers
pinned onto the fork). From here the workflow is yours to edit, version, and
tune; fetch and act on it with `goodeye workflows get <id-or-name>`. See
[Workflows](workflows.md) for teaching and optimizing a workflow against its
verifiers.

## Step 6: Author your own workflow

Forking adapts someone else's work; you can also author a workflow from scratch.
A workflow is markdown with a short metadata header: `name`, `description`, and
`outcome` are required, and `tags` are optional. For agent-generated output, pipe
the body from stdin so no intermediate file is left behind:

```sh
goodeye workflows publish - \
  --name high-signal-chart \
  --description "Produce a publication-quality chart on a topic, gated by a design verifier." \
  --outcome "Engagement on the charts we publish" \
  --tag data \
  --tag viz <<'EOF'
# Body

The runbook goes here: find an authoritative dataset, draft chart variants,
render the chart, then run the design verifier, revising until it passes. Inline
structural and functional checks belong here as fenced code blocks; reference any
semantic verifiers by their id.
EOF
```

When you already have a markdown file (with YAML front matter for the metadata),
publish it directly:

```sh
goodeye workflows publish ./high-signal-chart.md
```

Workflows are always private to you. Publishing the same `name` again appends a
new version. To share a workflow publicly later, claim a handle
(`goodeye me claim-handle <handle>`) and run
`goodeye templates publish <workflow-id-or-name>` as a separate, explicit step.

**Tip:** To design a workflow and its verifiers interactively, sign in and run
`goodeye design`, then pipe the printed prompt into your AI assistant.

**Already have skills on disk?** If you keep agent skills under
`~/.claude/skills/` (or `~/.agents/skills/`, `~/.cursor/skills/`), import one by
pointing `publish` at its directory and supplying an `outcome`. See
[Importing an existing skill](workflows.md#importing-an-existing-skill).

## Next steps

- [Workflows](workflows.md): version, teach, and optimize a workflow.
- [Verifiers](verifiers.md): add structural, functional, and semantic checks.
- [Templates](templates.md): publish and manage public templates.
- [Auditing workflows](auditing-workflows.md): grade a workflow against the
  best-practice checks.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, usage, and
  credits (`goodeye usage`).
