# Overview

Goodeye turns the business outcomes you care about into verified AI workflows
that agents run reliably. This page is the mental model: the problem Goodeye
solves, the chain it builds, and the pieces you will work with. Read it first,
then jump to [Getting Started](getting-started.md) for a hands-on first run.

## The problem

Capability is no longer the bottleneck. Frontier models can already do
remarkable work; the hard part is steering them toward a result you can measure,
and getting that result every time. Models are jagged: reliable where the output
is easy to check (code that compiles and passes its tests) and shaky where it is
not (brand voice, visual taste, pedagogy, policy). The capabilities that improved
fastest are the ones that were easy to verify, which leaves the work most
businesses actually need sitting in the hard-to-verify zone.

Observability tools tell you what an agent did after the fact. They do not keep
it on the result you care about while it works. That is the gap Goodeye closes:
reliable checks in the domains where "good" is a judgment call, so an agent stays
aligned to the outcome even where the model is weakest.

## What Goodeye is

You start from a business outcome, capture the work that moves it as a markdown
runbook (a workflow), and pair that runbook with checks (verifiers) that score an
AI agent's output against a measurable result. The agent runs the workflow, the
verifiers judge, and the agent revises until the output passes.

The intended caller is an AI agent acting on your behalf, and it runs a workflow
rather than just reading it. That behavior is the agent contract, and most of
Goodeye is built around it (see [The agent contract](#the-agent-contract) below).

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
  agent's output to keep it on the outcome-aligned path.

A holistic "is this output good overall?" check is not a Goodeye verifier. Every
verifier targets a specific outcome and a specific failure mode.

## The agent contract

The single most important behavior to internalize: when an agent fetches a
workflow or template body, it executes that body as your runbook. It does not
summarize the steps or print them for you to follow. A workflow can call tools
and verifiers along the way; those are the agent's hands and quality gates, and
the workflow is how the agent knows what to do with them.

```diagram-agent-loop
Agent loads the workflow | the fetched body is the runbook
^ Executes the steps | calls tools and verifiers
Verifiers judge the output | pass or fail with reasoning
exit: Passes, ship the result
loop: Fails, revise and re-run until verifiers pass
```

## Workflow (private) vs template (public)

| Aspect | Workflow (private) | Template (public) |
|---|---|---|
| Visibility | Private; shared only by grant | Public in the catalog |
| Mutability | Editable in place | Immutable snapshot |
| New version | On each save | On each publish |
| Who can read | Owner and grantees | Anyone, fully public |

A **workflow** is the private stored object: a markdown runbook with a `name`, a
one-line `description`, a declared `outcome`, and optional `tags`. Workflows are
private to you by default. You can share one privately with named users or teams
through a grant (see [Teams](teams.md)), but a workflow never becomes public on
its own.

A **template** is the public form of a workflow. To share publicly, you publish a
snapshot of a workflow as a template version under your handle. Templates are
immutable and versioned: continued edits to your private workflow never leak into
a published template, and a new round of work becomes a new version. Anyone (and
any agent) can find a template, fetch it, and run it directly. To get a saveable,
editable copy of their own, an authenticated user **forks** the template into a
new private workflow that carries lineage back to the version it came from.

Non-owner reads of a template carry an unverified-template safety banner as a
cross-user trust signal. Private workflows carry no banner, because every reader
already has explicit access.

See [Workflows](workflows.md) and [Templates](templates.md) for the full
lifecycle.

## Verifiers at a glance

A verifier is a check the workflow runs on agent output. It returns pass or fail
with reasoning. There are three types, and all three can coexist in one workflow:

- **Structural**: format, schema, required fields, presence. Lives inline in the
  workflow body. Deterministic and free.
- **Functional**: tests, numeric bounds, regex, hashes, and similar programmatic
  checks. Also inline. Deterministic and free.
- **Semantic**: interpretive judgment (tone, factuality, image quality) by an LLM
  judge calibrated with example pass and fail cases. Deployed once and referenced
  from the workflow by id.

Semantic verifiers are where Goodeye earns its keep, because they bring a
reliable check to outputs that are not obviously right or wrong. Image and
multimodal outcomes are a natural fit: a semantic verifier can score a generated
image against the result you want, the same way it scores text. See
[Verifiers](verifiers.md) and [Image Generators](image-generators.md).

## Improving a workflow against its outcome

Saving a workflow is the start, not the finish. Because every workflow is tied to
a measurable outcome and gated by verifiers, you can improve it against real
results over time:

```diagram-steering-loop
Design and save | author the workflow and its verifiers
^ Teach and optimize | fold in real-run feedback, tune against the verifiers
Audit against the checks | met, or gaps to fix
exit: Checks met, ship and publish
loop: Gaps found, teach and optimize again
```

- **Design** a workflow and its verifiers interactively, then save it.
- **Teach** it by running it on real inputs and folding your reactions back in.
- **Optimize** it automatically against its own verifier outcomes.
- **Audit** it against a best-practice rubric to find and fix gaps.

This loop is what makes Goodeye an outcome-alignment tool rather than a place to
store runbooks. See [Workflows](workflows.md) and
[Auditing workflows](auditing-workflows.md).

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
| Author, version, teach, and optimize a workflow | [Workflows](workflows.md) |
| Add structural, functional, and semantic checks | [Verifiers](verifiers.md) |
| Grade a workflow against the best-practice checks | [Auditing workflows](auditing-workflows.md) |
| Publish, fork, and manage public templates | [Templates](templates.md) |
| Generate images inside a workflow | [Image Generators](image-generators.md) |
| Host and serve images with durable URLs | [Images](images.md) |
| Share workflows with teammates | [Teams](teams.md) |
| Manage handles, API keys, usage, and credits | [Accounts and Billing](accounts-and-billing.md) |
| Connect over the command line, MCP, or REST | [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md) |
