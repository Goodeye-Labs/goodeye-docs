# Getting Started

This guide takes a skill file sitting on one of your machines and turns it into a
hosted skill: private to you, current on every machine and agent you run, and
shareable with a teammate in one command. Then it adds a hosted check and shows
how to author a skill from scratch. For the concepts behind each step, see
[Overview](overview.md).

If you would rather try Goodeye before making an account, skip to
[Running a public template](#running-a-public-template), which needs no sign-in.

```diagram-first-run
Install the CLI | uv tool install goodeye
Sign in | goodeye register
Bring a skill you already have | one publish per skill file
Sync it everywhere | every machine reads the current version
* Share it | a teammate runs your exact skill and checks
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

## Step 2: Sign in

Hosting skills of your own needs an account. At a machine with a browser:

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

## Step 3: Bring a skill you already have

If you keep agent skill files on disk, you already have what you need. A skill
file is a directory holding a `SKILL.md` plus optional siblings, which is exactly
what `publish` expects, so importing one is a single command:

```sh
goodeye skills publish ~/.claude/skills/high-signal-chart
```

`SKILL.md` becomes the hosted skill's body and the sibling files upload with it.
Front-matter keys Goodeye does not recognize are preserved verbatim. The command
prints the skill's name and new version number, its `skill_id`, and a
`version_token` you keep for the next update.

The same works for `~/.agents/skills/` and `~/.cursor/skills/`, or any directory
in that shape. To bring over a library, run one `publish` per skill file. See
[Importing a skill file from disk](skills.md#importing-a-skill-file-from-disk).

No skill files yet? Skip to [Step 7](#step-7-author-a-skill-from-scratch) and
author one, or fork a public template in
[Running a public template](#running-a-public-template).

Confirm what landed:

```sh
goodeye skills list
```

Your skill is private from this moment. Nothing is public until you publish a
template, which is a separate step covered in [Templates](templates.md).

## Step 4: Sync it to every machine and agent

The hosted skill is now the source of truth. Point Goodeye at the directories
your tools read, and it mirrors your skills into each one:

```sh
goodeye skills sync target add --preset claude    # ~/.claude/skills
goodeye skills sync target add --preset agents    # ~/.agents/skills (Codex reads here too)
goodeye skills sync target add --preset cursor    # ~/.cursor/skills
```

Any other directory works too; pass a path instead of a preset. Then pull:

```sh
goodeye skills sync
```

Each target now holds `<slug>/SKILL.md` plus siblings for every skill you own.
Run the same two commands on your other machines and they all read the same
current version.

One case to know about on this first pull: the skill you published in Step 3 came
from a directory that is now a sync target, so a copy of it is already sitting
there. Goodeye did not write that copy, so it reports the skill as modified and
leaves it alone rather than overwriting work it does not recognize. The hosted
version is what you just published, so adopt it once:

```sh
goodeye skills sync pull --force <slug>
```

From then on it is tracked like everything else and updates on every pull. This
applies only to skills already on disk before the target was added; anything you
publish from elsewhere lands normally.

From here, editing the hosted skill is what updates everything. Change it once,
and every machine and every agent picks it up. Now that you have a target,
automatic sync is on, so this happens without you asking: after a command
finishes, the CLI brings down new and updated skills in the background, no more
than once an hour.

Automatic sync is conservative. It never overwrites your local edits, never
deletes skill files, and never pushes. A local conflict is reported rather than
clobbered. It is suppressed in CI, for machine-readable output, and during a
manual sync. Turn it off with `goodeye skills sync auto off` and it stays off,
however many targets you add later.

Edited a skill file on disk and want the hosted copy to match? Use
`goodeye skills sync push`. Check for drift at any time with
`goodeye skills sync status`. See
[Syncing a bundle locally](skills.md#syncing-a-bundle-locally).

## Step 5: Share it privately

Grant a named user or team access. Identify them by `@handle`, email address, or
UUID; a teammate who has not claimed a handle yet is reachable by email:

```sh
goodeye skills grant high-signal-chart @teammate view
```

Roles are `view` (fetch and run), `edit` (also save new versions), and
`admin`. The grantee's agent now runs your skill, and any semantic verifiers it
references travel with the grant at the same version, so their output is held to
the checks you wrote. When you improve the skill, they get the improvement on
their next pull. There is no copy of it to go stale.

Review and revoke the same way:

```sh
goodeye skills grants high-signal-chart
goodeye skills revoke-grant high-signal-chart @teammate
```

A grant reaches a person or a whole team. Your own headless machines need no
grant: authenticated with your own API key, they already read every skill you
own. See [Teams](teams.md) for team-wide grants and invitations, and
[Sharing with grants](skills.md#sharing-with-grants) for the full model.

## Step 6: Add a hosted check

A skill says what to do. A verifier says whether the result is acceptable.
Structural and functional checks live inline in the skill body and cost nothing
to run. Interpretive checks (tone, factuality, chart quality) are semantic
verifiers: deploy one once, and every skill that references it runs that exact
version.

A semantic verifier is a JSON object holding a criterion and a couple of
calibration examples:

```sh
goodeye verifiers deploy ./claims-cite-source.json
```

It prints a `verifier_id`, a `version`, and a `version_token`. Reference the id
from your skill body, and the agent runs the check on its own output and revises
until it passes. Because the verifier is hosted rather than pasted into the
skill, everyone you granted the skill to runs the same check at the same version.

Keep each verifier specific. "Every factual claim is backed by the provided
source" is a verifier. "Is this good?" is not. See [Verifiers](verifiers.md) for
the payload shape, versioning, and the three check types.

## Step 7: Author a skill from scratch

The best path is a guided design session. It works with you to design a skill and
the verifiers that gate it, then saves the result when you approve it:

```sh
goodeye design
```

`goodeye design` prints a designer prompt; pipe it into your AI assistant
(`goodeye design > design.md`, or straight into your agent) and follow along.

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

Publishing the same `name` again appends a new version.

## Running a public template

A template is the public form of someone's skill, addressable as `@handle/slug`.
Running one needs no account, which makes it the quickest way to watch the loop
work. Browse the catalog:

```sh
goodeye templates list
```

Point your agent at one and tell it to run the template:

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

To read the runbook your agent executes, fetch it by hand:

```sh
goodeye templates get @randalolson/high-signal-chart-workflow
```

The body opens with a standing directive telling the calling agent to run the
skill on your behalf rather than display it, to quote each verifier's real
verdict, and to tell you at the end how the output was checked. Pass
`--output PATH` for the raw markdown without it, or `--json` for the full record.

Like what it produced? Fork it into a private skill you own and can edit. This is
the one step in this section that needs an account, since the fork lands in your
registry:

```sh
goodeye templates fork @randalolson/high-signal-chart-workflow
```

The fork carries lineage back to the version it came from, and any semantic
verifiers pinned onto it come along. From there it syncs and shares like any
other skill you own.

**Notes**

- **Credits:** the anonymous run draws on a small monthly grant for anonymous
  use. That grant covers Goodeye-metered work (the verifier run above), not
  your agent's own model usage, which bills through
  whatever model you run it on. See
  [Accounts and Billing](accounts-and-billing.md).
- **Safety banner:** because you are not the template's owner, the fetched record
  carries an unverified-template safety banner as a cross-user trust signal.

## Connecting a different surface

This guide uses the CLI. The same operations exist over MCP and REST:

- **A chat or connector client** (Claude.ai, Claude Desktop, Claude Code, Codex,
  Cursor, VS Code, Windsurf) connects over MCP at `https://mcp.goodeye.dev/mcp`,
  where the Goodeye tools appear natively. See [MCP](mcp.md).
- **A service or integration** calls the REST API at `https://api.goodeye.dev/v1`
  with an API key. See [REST API](rest-api.md).

## Next steps

- [Skills](skills.md): version, teach, optimize, and sync skills.
- [Teams](teams.md): share with a whole team and manage invitations.
- [Verifiers](verifiers.md): add structural, functional, and semantic checks.
- [Templates](templates.md): publish and manage public templates.
- [Auditing skills](auditing-skills.md): grade a skill against the authoring
  checks.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, usage, and
  credits (`goodeye usage`).
