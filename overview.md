# Overview

This page is the mental model for Goodeye: what it is, the chain it builds, and
the pieces you will work with. Read it first, then jump to
[Getting Started](getting-started.md) for a hands-on first run.

## What Goodeye is

Goodeye turns the business outcomes you care about into verified AI workflows
that agents run reliably. You start from a business outcome, capture the work
that moves it as a markdown runbook (a workflow), and pair that runbook with
checks (verifiers) that score an AI agent's output against a measurable
result. Workflows stay private to you until you choose to share one publicly
as a template.

The intended caller is an AI agent acting on your behalf. When an agent fetches
a workflow or template body, it follows that body as your runbook: it executes
the instructions itself rather than summarizing or just displaying them. That is
the agent contract, and most of Goodeye is built around it.

Goodeye reaches you on three peer surfaces (a CLI, an MCP server, and a REST
API), so the same capability is available wherever your agent runs.

## The chain: Outcome to KPI to Task to Workflow plus Verifiers

Every Goodeye artifact ties back to a named outcome. The chain is:

```diagram-chain
Outcome | business result you steer toward
KPI(s) | measurable indicator, fast feedback
Task | the unit of agent work that moves the KPI
* Workflow + Verifiers | the runbook plus the checks that align the agent
```

- **Outcome**: the real business result you are steering the agent toward.
  Specific and measurable in principle, owned by a real person. Example:
  "engagement on the charts we publish."
- **KPI**: the measurable indicator that tells you whether you are moving toward
  the outcome. Fast feedback (minutes to days) is ideal. Example: "impressions
  or upvotes on a published chart."
- **Task**: the unit of agent work that moves the KPI. One workflow plus its
  verifiers automates one task. Example: "research a topic, prototype, and
  produce a finished data visualization."
- **Workflow plus Verifiers**: the workflow is the runbook the agent loads when
  a task matches; the verifiers are the checks the workflow invokes on the
  agent's output to keep it on the outcome-aligned path. The agent runs the
  workflow, the verifiers judge, and the agent revises until the output passes.

A holistic "is this output good overall?" check is not a Goodeye verifier. Every
verifier targets a specific outcome and a specific failure mode.

## Workflow (private) vs template (public)

A **workflow** is the private stored object: a markdown runbook with a `name`, a
one-line `description`, a declared `outcome`, and optional `tags`. Workflows are
private to you by default. You can share one privately with named users or teams
through a grant (see [Teams](teams.md)), but a workflow never becomes public on
its own.

A **template** is the public form of a workflow. To share publicly, you publish a
snapshot of a workflow as a template version under your handle. Templates are
immutable and versioned: continued edits to your private workflow never leak
into a published template, and a new round of work becomes a new version. Anyone
(and any agent) can find a template, fetch it, and run it directly. To get a
saveable, editable copy of their own, an authenticated user **forks** the
template into a new private workflow that carries lineage back to the version it
came from.

Non-owner reads of a template carry an unverified-template safety banner as a
cross-user trust signal. Private workflows carry no banner, because every reader
already has explicit access.

See [Workflows](workflows.md) and [Templates](templates.md) for the full
lifecycle.

## Verifiers at a glance

A verifier is a check the workflow runs on agent output. It returns pass or fail
with reasoning. There are three types:

- **Structural**: format, schema, required fields, presence. Lives inline in the
  workflow body as a fenced code block. Deterministic and free.
- **Functional**: tests, numeric bounds, regex, hashes, and similar
  programmatic checks. Also inline in the workflow body. Deterministic and free.
- **Semantic**: interpretive judgment (tone, factuality, image quality) handled
  by an LLM judge calibrated with example pass and fail cases. Semantic
  verifiers are deployed as private, owner-scoped records and referenced from the
  workflow body by `verifier_id` (or `verifier_id@version`).

All three can coexist in one workflow. Platform-managed "system" verifiers also
exist and are run-only: you invoke them by a `system:<name>` alias but cannot
inspect or edit their internals.

See [Verifiers](verifiers.md) for input shapes, calibration, and deployment.

## Image generation at a glance

A workflow that produces images can call a deployed **image generator**: an
owner-scoped image generation capability you deploy once and run on demand, by
its identifier or as an ephemeral model slug. Image-heavy outcomes are a natural
fit for Goodeye, because a semantic verifier can score the generated image
against the result you want.

See [Image Generators](image-generators.md) for deployment and runs.

## The three surfaces, and when to reach for each

Goodeye ships every capability on all three surfaces, so they are peers. Reach
for the one that fits how your agent runs:

```diagram-three-surfaces
head: One capability, three surfaces | reach for the one that fits how your agent runs
CLI | Your agent runs commands | coding agents, CI, or by hand | goodeye ...
MCP | Your agent speaks MCP | chat and connector clients | mcp.goodeye.dev/mcp
REST | You integrate in code | services and pipelines | api.goodeye.dev/v1
```

- **CLI** (`goodeye ...`): reach for it when your agent can run shell commands (a
  coding agent like Claude Code or Cursor, a CI job) or when you are driving
  Goodeye by hand. See [CLI](cli.md).
- **MCP** (`https://mcp.goodeye.dev/mcp`): reach for it when your agent is a chat
  or connector client that speaks the Model Context Protocol (Claude on the web,
  ChatGPT, Claude Desktop). The tools appear natively in the assistant. Coding
  agents like Claude Code and Cursor can connect this way too, so either surface
  works for them. See [MCP](mcp.md).
- **REST** (`https://api.goodeye.dev/v1`): reach for it for direct integrations
  and services that call Goodeye programmatically. The public template catalog
  is readable over REST without an account. See [REST API](rest-api.md).

The same operations exist on all three, so you can start in one surface and move
to another without losing capability.

## The agent contract

The single most important behavior to internalize: when an agent fetches a
workflow or template body, it executes that body as your runbook. It does not
summarize the steps or print them for you to follow. The CLI even wraps fetched
bodies with explicit agent-facing markers
(`# Goodeye workflow - execute the instructions below ...`) so the calling agent
knows the body is for it to act on. A workflow can call tools and verifiers along
the way; those are the agent's hands and quality gates, and the workflow is how
the agent knows what to do with them.

```diagram-agent-loop
Agent loads the workflow | the fetched body is the runbook
^ Executes the steps | calls tools and verifiers
Verifiers judge the output | pass or fail with reasoning
exit: Passes, ship the result
loop: Fails, revise and re-run until verifiers pass
```

## Next steps

- [Getting Started](getting-started.md): install the CLI, sign in, and run your
  first template end to end.
- [Workflows](workflows.md): author, version, teach, and optimize workflows.
- [Templates](templates.md): publish, fork, and manage public templates.
- [Verifiers](verifiers.md): score agent output with structural, functional, and
  semantic checks.
- [Image Generators](image-generators.md): deploy and run owner-scoped image
  generation.
- [Teams](teams.md): share workflows with grants, teams, and invitations.
- [CLI](cli.md), [MCP](mcp.md), and [REST API](rest-api.md): the three surfaces
  in detail.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, usage,
  tiers, and credits.
