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

## Three states per check

Each authoring check is in one of three states:

- **Met:** the check was assessed and every part of it passed.
- **Not met:** the check was assessed and at least one part fell short.
- **Not checked:** the check was not assessed.

The catalog shows all four checks on every template, including the ones a
workflow has not earned.

## How the checks are computed

The checks are computed once, when you publish a template version, and pinned to
that version. Publishing a new version recomputes them; older versions keep the
result they were published with.

Authoring checks never block publishing. Only the Safety check can stop a publish;
a template that earns no authoring checks still publishes. Publishing reports the
full breakdown so you can see exactly what to improve.

## The rubric behind the checks

The three authoring checks roll up eight underlying criteria. Each check is
earned only when all of its criteria pass.

**Outcome**

1. *Outcome quality:* the declared outcome names a specific, measurable result
   the workflow moves, not a restated task or a vague aspiration.
2. *Verifier outcome-specificity:* the referenced verifiers measure that outcome
   with explicit pass and fail conditions, and deterministic rules use
   programmatic checks rather than a judgment call.
3. *Success-criteria clarity:* the body defines what a correct output looks like
   in checkable terms.

**Runnable**

4. *Instructional actionability:* the steps read as clear, ordered, executable
   instructions with no contradictions.
5. *Input and output contract clarity:* it is unambiguous what the workflow
   consumes and what shape of output it produces.

**Well-formed**

6. *Discovery and coherence:* the description says what the workflow does and when
   to use it, terminology stays consistent, and the body stays on its outcome.
7. *Structural completeness:* an outcome is declared, at least one verifier is
   referenced, discovery tags are present, verifier references are pinned to a
   version, and reference links resolve.
8. *Authoring hygiene:* the body is lean (long detail moves into reference files),
   carries no stray formatting, and every referenced semantic verifier includes
   calibration examples.

## Running an audit

An audit is an opinionated health check against the rubric above. It returns a
prompt pack your agent runs locally, inspects the workflow body and every
directing sibling file, and produces a priority-ranked report: each finding is
tagged P0 (blocker), P1 (recommended), or P2 (polish), quotes the text to
change, and states one concrete fix. Requires at least `view` access.

You choose which findings to fix; the audit applies only what you approve and
never auto-saves, editing a local copy first when one exists and marking the
saved version `source=audit`. You can also audit a local skill that is not on
Goodeye yet, in which case the report recommends saving it.

- **CLI:** `goodeye workflows audit <id-or-name>` (omit the id to audit a local skill)
- **MCP tool:** `audit_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/audit`, or `GET /v1/audit/workflow-prompt` for a local skill

The audit is the place to improve a workflow before or after you publish it.

## Checking safety on demand

You can run the platform safety checks on demand, without publishing, against
either a saved workflow or a published template version. Each call makes no
changes, bills two verifier runs, and returns a `status` of `clean`, `flagged`,
`blocked`, or `error`.

A saved workflow previews what a
[template publish](templates.md#publishing-a-version) would compute:

- **CLI:** `goodeye workflows check-safety <id-or-name>` (`--version`, `--json`)
- **MCP tool:** `check_workflow_safety`
- **REST:** `POST /v1/workflows/{id_or_slug}/safety-check`

A published template version can be re-checked by anyone. Auth is optional: pass
`--anonymous` to run without sending an API key (anonymous spend bills against a
small per-caller credit budget). Unlike `get_template`, full reasoning is
returned regardless of ownership, because the caller paid for the runs:

- **CLI:** `goodeye templates check-safety <identifier>` (`--version`,
  `--anonymous`, `--json`)
- **MCP tool:** `check_template_safety`
- **REST:** `POST /v1/templates/{template_ref}/safety-check`

The on-demand template check covers the body, description, and outcome only; it
does not re-scan sibling files, and an over-long field is shortened for the scan
(the response flags that the scan was partial). The authoritative, durable trust
signal is the safety status that publishing computes over the whole bundle and
stores on the version.

## See also

- [Workflows](workflows.md): designing and saving a workflow, plus the teach and
  optimize tools that change one.
- [Templates](templates.md): publishing, the unverified-template banner, and how
  the checks appear to a reader.
- [Verifiers](verifiers.md): the semantic checks the Outcome check rewards.
