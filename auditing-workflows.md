# Auditing workflows

An audit grades a [workflow](workflows.md) against Goodeye's authoring checks:
the same checks every public [template](templates.md) displays. Run it before you
publish to see what a reader will see, or any time after to improve a workflow
you already shipped.

The checks describe how a workflow is written, not the outputs it produces. They
are not a guarantee of safety, authorship, accuracy, or fitness for any task. A
reader still has to decide whether a template fits their need.

## The authoring checks

Every public template shows four named checks: **Safety**, **Outcome**,
**Runnable**, and **Well-formed**. Together they give a reader an honest,
at-a-glance signal of how well a template's workflow is authored.

- **Safety** is the safety screen that runs before a template is published. See
  [Templates](templates.md#the-unverified-template-banner) for the safety states
  (clean, flagged, and not safety-checked) a reader sees and what they mean.
- **Outcome** means the workflow declares a measurable outcome and references
  [verifiers](verifiers.md) that check the output against it. This is the
  Goodeye difference: a workflow that knows what result it is moving and checks
  for it.
- **Runnable** means the workflow has clear, ordered instructions and a defined
  input and output contract an agent can follow.
- **Well-formed** means the workflow is discoverable, structurally complete, and
  cleanly authored.

## Check states

The three authoring checks (Outcome, Runnable, Well-formed) each read as one of:

- **Met:** the check was assessed and every part of it passed.
- **Not met:** the check was assessed and at least one part fell short.
- **Not checked:** the check was not assessed.

Safety is a separate badge with its own status (clean, flagged, or not checked);
see [Templates](templates.md#the-unverified-template-banner). Every public
template page shows all four checks, including the ones a workflow has not
earned.

## How the checks are computed

The checks are computed once, when you publish a template version, and pinned to
that version. Publishing a new version recomputes them; older versions keep the
result they were published with.

Authoring checks never block publishing. Only the Safety check can stop a publish;
a template that earns no authoring checks still publishes. Publishing reports the
full breakdown so you can see exactly what to improve.

## Running an audit

An audit is an opinionated health check against these checks. It returns a
prompt pack your agent runs locally, inspects the workflow body and every
directing sibling file, and produces a priority-ranked report: each finding is
tagged P0 (blocker), P1 (recommended), or P2 (polish), quotes the text to
change, and states one concrete fix. Requires at least `view` access.

You choose which findings to fix; the audit applies only what you approve and
never auto-saves, editing a local copy first when one exists and marking the
saved version `source=audit`. You can also audit a local workflow file not yet
saved to Goodeye, in which case the report recommends saving it.

- **CLI:** `goodeye workflows audit <id-or-name>` (omit the id to audit a local workflow file)
- **MCP tool:** `audit_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/audit`, or `GET /v1/audit/workflow-prompt` for a local workflow file

The audit is the place to improve a workflow before or after you publish it.

## Checking safety on demand

You can run the platform safety checks without publishing, against a saved
workflow or a published template version. The call makes no changes and runs the
safety checks (billed as verifier runs), returning a `status` of `clean`,
`flagged`, `blocked`, or `error`. What safety means, the states, and the banner
a reader sees are canonical in
[Templates](templates.md#the-unverified-template-banner); publishing computes the
authoritative status over the whole bundle and stores it on the version.

- Saved workflow: CLI `goodeye workflows check-safety <id-or-name>`, MCP
  `check_workflow_safety`, REST `POST /v1/workflows/{id_or_slug}/safety-check`.
- Published template version (re-checkable by anyone; pass `--anonymous` to run
  without an API key, billed against a small per-caller credit budget): CLI
  `goodeye templates check-safety <identifier>`, MCP `check_template_safety`,
  REST `POST /v1/templates/{template_ref}/safety-check`.

## See also

- [Workflows](workflows.md): designing and saving a workflow, plus the teach and
  optimize tools that change one.
- [Templates](templates.md): publishing, the unverified-template banner, and how
  the checks appear to a reader.
- [Verifiers](verifiers.md): the semantic checks the Outcome check rewards.
