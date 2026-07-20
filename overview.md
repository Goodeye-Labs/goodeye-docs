# Overview

Goodeye is a private home for the skills your AI follows and the verifiers its
work must pass. A skill is a markdown runbook. A verifier is a check the output
has to clear. You keep both in one account, they stay private until you say
otherwise, and they reach every machine and every agent you run.

This page is the mental model. For a hands-on first run, see
[Getting Started](getting-started.md).

## Why host them at all

Most people start with skill files on disk, under `~/.claude/skills/` or
whatever directory their tool reads. That works until it doesn't:

- You work on a laptop and a desktop, and the good version is on the other one.
- You run more than one agent, and each keeps its own copy.
- A teammate asks for your skill, so you paste it into chat. Now two copies
  exist and only yours ever gets fixed.
- The standard that makes the skill worth running is in your head, not in the
  file, so nobody else's output clears it.

Goodeye is where those live instead. Your skills and verifiers sit in one
account you own, and three things follow from that.

## What you get

### Private by default

Skills are private. Nothing is public until you publish a template, which is a
separate step you take on purpose. To share without going public, grant a named
user or team access, and the verifiers the skill references go with it at the
same version. Revoking works the same way. See
[Sharing with grants](skills.md#sharing-with-grants).

### Always in sync

The hosted skill is the source of truth. `goodeye skills sync` mirrors it down
into the directories your tools already read, so Claude Code, Codex, Cursor, and
anything else that loads skill files from disk all read the same current
version. Automatic sync is on once you have a target, unless you turn it off, so
edit a skill once and every machine picks it up on its own. Nobody has to be
told to update. See
[Syncing a bundle locally](skills.md#syncing-a-bundle-locally).

### Verifiers, hosted

A verifier scores one concrete property of the output. It is never a vague "is
this good overall?" rating. Deploy a semantic verifier once and every skill that
references it runs that exact version, on your laptop, in CI, or on the machine
of someone you granted it to. See [Verifiers](verifiers.md).

```diagram-chain
Skill | the markdown runbook your agent follows
Verifiers | the checks its output has to clear
Grant | who else can run them, at the same version
* In sync everywhere | every machine and agent runs the current one
```

## The pieces

- **Skill**: a markdown runbook stored privately in your Goodeye account, with a
  `name`, a one-line `description`, and optional `tags`. An agent fetches the body
  and executes it as a runbook.
- **Verifier**: a check the skill runs on agent output. Structural, functional, or
  semantic (an LLM judge calibrated with examples).
- **Grant**: private access to one of your skills for a named user or team. The
  verifiers the skill references travel with it.
- **Sync target**: a local directory Goodeye mirrors your hosted skills into, so
  the agents on that machine read the current version.
- **Template**: the public form of a skill, shared under your handle so other
  people and their agents can find, fetch, and fork it.
- **Image generator**: a deployed, owner-scoped image generation capability a
  skill can call.
- **Hosted image**: an image stored by Goodeye with a stable URL that never
  changes, including images produced by a generator.

## The agent contract

The single most important behavior to internalize: when an agent fetches a skill
or template body, it executes that body as your runbook. It does not summarize the
steps or print them for you to follow. A skill can call tools and verifiers along
the way; those are the agent's hands and quality gates, and the skill is how the
agent knows what to do with them.

```diagram-agent-loop
Agent loads the skill | the fetched body is the runbook
^ Executes the steps | calls tools and verifiers
Verifiers judge the output | pass or fail with reasoning
exit: Passes, ship the result
loop: Fails, revise and re-run until verifiers pass
```

## Skill (private) vs template (public)

| Aspect | Skill (private) | Template (public) |
|---|---|---|
| Visibility | Private; shared only by grant | Public in the catalog |
| Mutability | Editable in place | Immutable snapshot |
| New version | On each save | On each publish |
| Who can read | Owner and grantees | Anyone, fully public |

A **skill** is the private stored object: a markdown runbook with a `name`, a
one-line `description`, and optional `tags`. Skills are private to you by default.
You can share one privately with named users or teams through a grant (see
[Teams](teams.md)), but a skill never becomes public on its own.

A **template** is the public form of a skill. To share publicly, you publish a
snapshot of a skill as a template version under your handle. Templates are
immutable and versioned: continued edits to your private skill never leak into a
published template, and a new round of work becomes a new version. Anyone (and any
agent) can find a template, fetch it, and run it directly. To get a saveable,
editable copy of their own, an authenticated user **forks** the template into a
new private skill that carries lineage back to the version it came from.

Non-owner reads of a template carry an unverified-template safety banner as a
cross-user trust signal. Private skills carry no banner, because every reader
already has explicit access.

See [Skills](skills.md) and [Templates](templates.md) for the full lifecycle.

## Verifiers at a glance

A verifier is a check the skill runs on agent output. It returns pass or fail with
reasoning. There are three types, and all three can coexist in one skill:

- **Structural**: format, schema, required fields, presence. Lives inline in the
  skill body. Deterministic and free.
- **Functional**: tests, numeric bounds, regex, hashes, and similar programmatic
  checks. Also inline. Deterministic and free.
- **Semantic**: interpretive judgment (tone, factuality, image quality) by an LLM
  judge calibrated with example pass and fail cases. Deployed once and referenced
  from the skill by id.

Semantic verifiers cover the outputs a plain test has nothing to grab onto: the
ones that are not obviously right or wrong. Image and multimodal work fits the
same way, since a semantic verifier can judge a generated image against the
result you want exactly as it judges text. See [Verifiers](verifiers.md) and
[Image Generators](image-generators.md).

## Improving a skill

Saving a skill is the start, not the finish. Because every skill is gated by
verifiers, you can improve it against real results over time:

```diagram-steering-loop
Design and save | author the skill and its verifiers
^ Teach and optimize | fold in real-run feedback, tune against the verifiers
Audit against the checks | met, or gaps to fix
exit: Checks met, ship and publish
loop: Gaps found, teach and optimize again
```

- **Design** a skill and its verifiers interactively, then save it.
- **Teach** it by running it on real inputs and folding your reactions back in.
- **Optimize** it automatically against its own verifier results.
- **Audit** it against the authoring checks to find and fix gaps.

A saved skill is a first draft. Teach, optimize, and audit are how it gets better
against real results. When you improve one, everyone you granted it to picks up
the improvement on their next pull. See [Skills](skills.md) and
[Auditing skills](auditing-skills.md).

## The three surfaces

Goodeye ships every capability on all three surfaces, so they are peers. Reach
for the one that fits how your agent runs:

```diagram-three-surfaces
head: One capability, three surfaces | reach for the one that fits how your agent runs
CLI | Your agent runs commands | coding agents, CI, or by hand | goodeye ...
MCP | Your agent speaks MCP | chat and connector clients | mcp.goodeye.dev/mcp
REST | You integrate in code | services and pipelines | api.goodeye.dev/v1
```

The same operations exist on all three, so you can start in one surface and move
to another without losing capability. The public template catalog is also
readable over REST without an account. [Getting Started](getting-started.md)
works through the CLI and points at the other two, and [CLI](cli.md),
[MCP](mcp.md), and [REST API](rest-api.md) are the per-surface references.

## Where to go next

| You want to... | Start here |
|---|---|
| Put a skill you already have into Goodeye | [Getting Started](getting-started.md) |
| Mirror your skills onto every machine you use | [Syncing a bundle locally](skills.md#syncing-a-bundle-locally) |
| Share a skill privately with a person or team | [Teams](teams.md) |
| Add structural, functional, and semantic checks | [Verifiers](verifiers.md) |
| Author, version, teach, and optimize a skill | [Skills](skills.md) |
| Grade a skill against the authoring checks | [Auditing skills](auditing-skills.md) |
| Publish, fork, and manage public templates | [Templates](templates.md) |
| Generate images inside a skill | [Image Generators](image-generators.md) |
| Host and serve images with durable URLs | [Images](images.md) |
| Manage handles, API keys, usage, and credits | [Accounts and Billing](accounts-and-billing.md) |
| Connect over the command line, MCP, or REST | [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md) |
