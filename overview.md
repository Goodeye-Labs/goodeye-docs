# Overview

Goodeye makes an AI agent meet your standard before you ever see the output,
even on work too subjective for a test. This page is the mental model: the
problem Goodeye solves, the pieces you will work with, and how they fit together.
Read it first, then jump to [Getting Started](getting-started.md) for a hands-on
first run.

## The problem

Capability is no longer the bottleneck. Frontier models can already do
remarkable work; the hard part is getting work you would sign your name to, every
time. Models are jagged: reliable where the output is easy to check (code that
compiles and passes its tests) and shaky where it is not (brand voice, visual
taste, pedagogy, policy). The capabilities that improved fastest are the ones
that were easy to verify, which leaves the work most businesses actually need
sitting in the hard-to-verify zone.

Observability tools tell you what an agent did after the fact. They do not hold
it to your standard while it works. That is the gap Goodeye closes: reliable
checks in the domains where "good" is a judgment call, so an agent gets the work
right even where the model is weakest.

## What Goodeye is

You capture the work you want done as a markdown runbook (a skill), and you pair
that runbook with checks (verifiers) that judge the agent's output. The agent
runs the skill, the verifiers judge, and the agent revises until the output
passes. Nothing reaches you until it has cleared your checks.

Skills are private. You share one with the people and teams you choose, and the
verifiers it references travel with it, so a teammate who runs your skill is held
to the same standard you are.

The intended caller is an AI agent acting on your behalf, and it runs a skill
rather than just reading it. That behavior is the agent contract, and most of
Goodeye is built around it (see [The agent contract](#the-agent-contract) below).

Goodeye reaches you on three peer surfaces (a CLI, an MCP server, and a REST
API), so the same capability is available wherever your agent runs.

## The pieces

```diagram-chain
Task | the recurring work you hand an agent
Skill | the markdown runbook the agent follows
Verifiers | the checks it must clear
* Verified output | nothing reaches you until the checks pass
```

- **Skill**: a markdown runbook stored privately in your Goodeye account, with a
  `name`, a one-line `description`, and optional `tags`. An agent fetches the body
  and executes it as a runbook.
- **Verifier**: a check the skill runs on agent output. Structural, functional, or
  semantic (an LLM judge calibrated with examples).
- **Template**: the public form of a skill, shared under your handle so other
  people and their agents can find, fetch, and fork it.
- **Image generator**: a deployed, owner-scoped image generation capability a
  skill can call.
- **Hosted image**: an image stored by Goodeye with a stable URL that never
  changes, including images produced by a generator.

A holistic "is this output good overall?" check is not a Goodeye verifier. Every
verifier targets one concrete property and one specific failure mode.

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
against real results. See [Skills](skills.md) and
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
walks through connecting each surface, and [CLI](cli.md), [MCP](mcp.md), and
[REST API](rest-api.md) are the per-surface references.

## Where to go next

| You want to... | Start here |
|---|---|
| Run your first public template, with no account | [Getting Started](getting-started.md) |
| Author, version, teach, and optimize a skill | [Skills](skills.md) |
| Add structural, functional, and semantic checks | [Verifiers](verifiers.md) |
| Grade a skill against the authoring checks | [Auditing skills](auditing-skills.md) |
| Publish, fork, and manage public templates | [Templates](templates.md) |
| Generate images inside a skill | [Image Generators](image-generators.md) |
| Host and serve images with durable URLs | [Images](images.md) |
| Share skills with teammates | [Teams](teams.md) |
| Manage handles, API keys, usage, and credits | [Accounts and Billing](accounts-and-billing.md) |
| Connect over the command line, MCP, or REST | [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md) |
