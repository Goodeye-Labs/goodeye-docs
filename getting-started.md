# Getting Started

This guide takes you from zero to a verified artifact: install the CLI, point
your agent at Goodeye, have it run a public template and watch a verifier return
PASS, then fork that template into your own private skill and author one from
scratch. No account is needed to run a template; you sign in when you want to
save work of your own. For the concepts behind each step, see
[Overview](overview.md).

```diagram-first-run
Install the CLI | uv tool install goodeye
Point your agent at Goodeye | CLI, MCP, or REST
Run a public template | your agent produces a verified artifact
Fork it into a private skill | a saveable, editable copy
* Author your own skill | from scratch, gated by your verifiers
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

Goodeye is built for an AI agent acting on your behalf. The agent fetches a skill
and executes it as a runbook (see
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
canonical demo, that is the high-signal chart template:

```text
Run the Goodeye template @randalolson/high-signal-chart-workflow.
```

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
the body to stdout:

```sh
goodeye templates get @randalolson/high-signal-chart-workflow
```

The body opens with a standing directive telling the calling agent to run the
skill on your behalf rather than display it, to quote each verifier's real
verdict, and to tell you at the end how the output was checked. That directive is
the agent contract in action. Pass `--output PATH` for the raw markdown without
it, or `--json` for the full record.

**Notes**

- **Credits:** the anonymous run draws on a small monthly grant for anonymous
  use. That grant covers Goodeye-metered work (the verifier run above and
  template safety checks), not your agent's own model usage, which bills through
  whatever model you run it on. See
  [Accounts and Billing](accounts-and-billing.md).
- **Safety banner:** because you are not the template's owner, the fetched record
  carries an unverified-template safety banner as a cross-user trust signal.

Ran it and like what it produced? Next you make it yours: sign in, then fork it
into a private skill you can edit and tune.

## Step 4: Sign in to save your work

Running templates needs no account. To fork a template or save a skill of your
own, create an account or sign in. The quickest path, at a machine with a
browser:

```sh
goodeye register   # new account
goodeye login      # existing account
```

Either command opens a verification URL on the hosted sign-in page, where you
continue with Google or email and approve in the browser; credentials are saved
locally. Confirm with `goodeye whoami`.

Agents, CI, and headless terminals can authenticate non-interactively with an
email-code flow instead. See [CLI](cli.md) for the `register-verify` /
`login-verify` steps, and [Accounts and Billing](accounts-and-billing.md) for
creating a `good_live_` API key for programmatic REST or MCP clients.

## Step 5: Fork the template into a private skill

Running a template is great for a one-off; to make it your own, fork it. Forking
copies the template into a new private skill you own, carrying lineage back to
the version you forked. This is the persistent, tunable path:

```sh
goodeye templates fork @randalolson/high-signal-chart-workflow
```

The command prints the new skill's id and slug (and any semantic verifiers
pinned onto the fork). From here the skill is yours to edit, version, and tune;
fetch and act on it with `goodeye skills get <id-or-name>`. See
[Skills](skills.md) for teaching and optimizing a skill against its verifiers.

## Step 6: Author your own skill

Forking adapts someone else's work; you can also author your own from scratch.
The best first path is a guided design session: it works with you to design a
skill and the verifiers that gate it, then saves the result when you approve it.

```sh
goodeye design
```

`goodeye design` prints a designer prompt; pipe it into your AI assistant
(`goodeye design > design.md`, or straight into your agent) and follow along. The
session produces the skill plus its verifiers and saves it for you.

Prefer to write it yourself? A skill is markdown with a short metadata header
(`name` and `description` are required; `tags` optional). Publish a file
directly, or pipe the body from stdin for agent-generated output so no
intermediate file is left behind:

```sh
goodeye skills publish ./high-signal-chart.md
# or, from stdin:
goodeye skills publish - \
  --name high-signal-chart \
  --description "Produce a publication-quality chart on a topic, gated by a design verifier." \
  --tag data --tag viz <<'EOF'
# Body: find an authoritative dataset, draft chart variants, render the chart,
# then run the design verifier, revising until it passes. Inline structural and
# functional checks go here as fenced code blocks; reference semantic verifiers
# by their id.
EOF
```

Skills are always private to you. Publishing the same `name` again appends a new
version. To share a skill publicly later, claim a handle
(`goodeye me claim-handle <handle>`) and run
`goodeye templates publish <skill-id-or-name>` as a separate, explicit step. To
share it privately with a teammate instead, grant them access; see
[Sharing with grants](skills.md#sharing-with-grants).

**Already have skill files on disk?** If you keep agent skill files under
`~/.claude/skills/` (or `~/.agents/skills/`, `~/.cursor/skills/`), import one by
pointing `publish` at its directory. See
[Importing a skill file from disk](skills.md#importing-a-skill-file-from-disk).

## Next steps

- [Skills](skills.md): version, teach, and optimize a skill.
- [Verifiers](verifiers.md): add structural, functional, and semantic checks.
- [Templates](templates.md): publish and manage public templates.
- [Auditing skills](auditing-skills.md): grade a skill against the authoring
  checks.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, usage, and
  credits (`goodeye usage`).
