# Auditing skills

An audit grades a [skill](skills.md) against Goodeye's authoring checks: the same
checks every public [template](templates.md) displays. Run it before you publish
to see what a reader will see, or any time after to improve a skill you already
shipped.

The checks describe how a skill is written, not the outputs it produces. They are
not a guarantee of safety, authorship, accuracy, or fitness for any task. A reader
still has to decide whether a template fits their need.

## The authoring checks

Every public template shows three named checks: **Safety**, **Runnable**, and
**Well-formed**. Together they give a reader an honest, at-a-glance signal of how
well a template's skill is authored.

```diagram-authoring-checks
head: Authoring checks | how a template's skill is written, shown to every reader
Safety | screened at publish | clean, flagged, or not checked | gate
Runnable | clear, ordered steps | defined inputs and outputs | earned
* Well-formed | discoverable and held to a standard | real, specific verifiers | earned
```

- **Safety** is the safety screen that runs before a template is published. See
  [Templates](templates.md#the-unverified-template-banner) for the safety states
  (clean, flagged, and not safety-checked) a reader sees and what they mean.
- **Runnable** means the skill has clear, ordered instructions and a defined input
  and output contract an agent can follow.
- **Well-formed** means the skill is discoverable, coherent, and held to a stated
  standard: a clear trigger description, consistent terminology, a definition of
  what a correct output looks like in checkable terms, and at least one real,
  specific [verifier](verifiers.md) rather than a holistic "is this good overall?"
  check.

## Check states

The two earned checks (Runnable and Well-formed) each read as one of:

- **Met:** the check was assessed and every part of it passed.
- **Not met:** the check was assessed and at least one part fell short.
- **Not checked:** the check was not assessed.

Safety is a separate badge with its own status (clean, flagged, or not checked);
see [Templates](templates.md#the-unverified-template-banner). Every public
template page shows all three checks, including the ones a skill has not earned.

## How the checks are computed

The checks are computed once, when you publish a template version, and pinned to
that version. Publishing a new version recomputes them; older versions keep the
result they were published with.

Authoring checks never block publishing. Only the Safety check can stop a publish;
a template that earns no authoring checks still publishes. Publishing reports the
full breakdown so you can see exactly what to improve, including non-blocking
authoring notes (a missing tag set, an overlong body, a referenced verifier with
no calibration) that never change a badge.

## Running an audit

An audit is an opinionated health check against these checks. It returns a prompt
pack your agent runs locally, inspects the skill body and every directing sibling
file, and produces a priority-ranked report: each finding is tagged P0 (blocker),
P1 (recommended), or P2 (polish), quotes the text to change, and states one
concrete fix. Requires at least `view` access.

You choose which findings to fix; the audit applies only what you approve and
never auto-saves, editing a local copy first when one exists. You can also audit a
skill file on disk that is not saved to Goodeye yet, in which case the report
recommends saving it as a hosted skill.

- **CLI:** `goodeye skills audit <id-or-name>` (omit the id to audit a skill file
  on disk)
- **MCP tool:** `audit_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/audit`, or `GET /v1/audit/skill-prompt`
  for a skill file on disk

The audit is the place to improve a skill before or after you publish it.

## Checking safety on demand

You can run the platform safety checks without publishing, against a hosted skill
or a published template version. The call makes no changes and runs the safety
checks (billed as verifier runs), returning a `status` of `clean`, `flagged`,
`blocked`, or `error`. What safety means, the states, and the banner a reader sees
are canonical in [Templates](templates.md#the-unverified-template-banner);
publishing computes the authoritative status over the whole bundle and stores it
on the version.

- Hosted skill: CLI `goodeye skills check-safety <id-or-name>`, MCP
  `check_skill_safety`, REST `POST /v1/skills/{id_or_slug}/safety-check`.
- Published template version (re-checkable by anyone; pass `--anonymous` to run
  without an API key, billed against a small per-caller credit budget): CLI
  `goodeye templates check-safety <identifier>`, MCP `check_template_safety`,
  REST `POST /v1/templates/{template_ref}/safety-check`.

## See also

- [Skills](skills.md): designing and saving a skill, plus the teach and optimize
  tools that change one.
- [Templates](templates.md): publishing, the unverified-template banner, and how
  the checks appear to a reader.
- [Verifiers](verifiers.md): the semantic checks the Well-formed check rewards.
