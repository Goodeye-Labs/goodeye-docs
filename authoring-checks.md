# What we check

Every public [template](templates.md) shows four named checks: **Safety**,
**Outcome**, **Runnable**, and **Well-formed**. They give a reader an honest,
at-a-glance signal of how well a template's [workflow](workflows.md) is authored.

The checks describe how the workflow is written, not the outputs it produces.
They are not a guarantee of safety, authorship, accuracy, or fitness for any
task. A reader still has to decide whether a template fits their need.

## The four checks

- **Safety** is the safety screen that runs before a template is published. See
  [Templates](templates.md#the-unverified-template-banner) for the safety states
  (clean, flagged, and not safety-checked) and what they mean.
- **Outcome** means the workflow declares a measurable outcome and references
  [verifiers](verifiers.md) that check the output against it. This is the
  Goodeye difference: a workflow that knows what result it is moving and checks
  for it.
- **Runnable** means the workflow has clear, ordered instructions and a defined
  input and output contract an agent can follow.
- **Well-formed** means the workflow is discoverable, structurally complete, and
  cleanly authored.

## Three states per check

Each authoring check is in one of three states, and the three are always kept
distinct:

- **Met:** the check was assessed and every part of it passed.
- **Not met:** the check was assessed and at least one part fell short.
- **Not checked:** the check was never assessed (for example a template version
  published before the checks existed). This is never the same as "not met".

The catalog shows all four checks on every template, so gaps are visible rather
than hidden.

## How the checks are computed

The checks are computed once, when you publish a template version, and pinned to
that version. Publishing a new version recomputes them; older versions keep the
result they were published with.

Authoring checks never block publishing. Only the Safety check can stop a publish;
a template that earns no authoring checks still publishes. The publish response
returns the full per-criterion breakdown so you can see exactly what to improve.

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
   carries no stray formatting, and every referenced semantic verifier ships
   calibration examples.

## Improving your checks

To see the full per-criterion breakdown for a workflow and apply targeted fixes,
run the audit. It reports against the same three checks and walks each finding
with a concrete fix, then applies only the changes you choose.

- **CLI:** `goodeye workflows audit <id>`
- **MCP tool:** `audit_workflow`
- **REST:** `POST /v1/workflows/{id}/audit`

The audit is the place to improve a workflow before or after you publish it. See
[Workflows](workflows.md) and [Verifiers](verifiers.md) for the authoring
practices the checks reward.
