# Verifiers

A verifier is a check a workflow runs on agent output to confirm the work hit a
measurable result. This page covers the three verifier types, how to deploy and
version your own semantic verifiers, and how workflows reference them.

A verifier is always outcome-specific. It scores one concrete property of the
output ("did the response cite a source for every claim?", "is the JSON valid
against this schema?"). It is never a holistic "is this good overall?" check.
If you cannot say what a pass and a fail look like in one sentence, the check is
not specific enough yet.

For where verifiers fit in the larger picture, see [Overview](overview.md). For
the runbooks that invoke them, see [Workflows](workflows.md).

## The three verifier types

A single workflow usually combines all three. They differ in what they check and
where they run.

### Structural

Format and schema checks: required fields are present, the output parses, the
shape matches. These are deterministic and live inline in the workflow body as
plain code or assertions. There is no registry object to deploy; the workflow
runs them itself.

Use a structural check when a pass or fail is decidable by inspecting the shape
of the output (valid JSON, all required keys present, the right number of rows).

### Functional

Tests, bounds, and computed checks: unit tests pass, a number lands inside a
range, a regular expression matches, a hash compares equal. Like structural
checks, functional checks are deterministic and live inline in the workflow
body. No registry object is involved.

Use a functional check when a pass or fail is decidable by running code against
the output (a test suite, a numeric tolerance, a checksum).

### Semantic

Interpretive judgments that need reading and reasoning, not just parsing: tone,
factuality against a source, pedagogical quality, whether an image matches a
brief. A semantic verifier is one judgment ("does this output satisfy this
criterion?") evaluated by an LLM judge, calibrated with labeled examples.

Unlike the other two types, a semantic verifier is a stored, versioned object in
the registry that you deploy, run, and reference by ID. The rest of this page is
about semantic verifiers.

**Note:** Semantic verifiers are private and owner-scoped. There is no public
verifier catalog. You deploy them for your own workflows, and access can cascade
to people you grant a workflow to.

## Semantic verifier concepts

When you deploy a semantic verifier you define three things:

- **Criterion**: the rubric the judge applies, written as a direct instruction
  (for example, "Return passed=true when every factual claim is supported by the
  provided source, otherwise passed=false"). This is where you draw the pass/fail
  line.
- **Calibration examples**: a handful of labeled examples (3 to 10 is typical),
  each marked as a pass or a fail, optionally with a short rationale. The judge
  sees these as demonstrations so its verdicts stay consistent with yours.
- **Input contract**: the shape of the inputs the judge reads at run time. One of:
  - `text`: one or more named text fields only.
  - `text_image`: named text fields plus one image.
  - `image`: a single image only (no text fields).

The contract you choose determines what `run_verifier` accepts. For `text` and
`text_image` you declare `input_fields` (the named text inputs); for `image` you
leave them empty. A run must supply exactly those fields, no more and no fewer,
and must include an image URL when the contract calls for one.

Each deploy also pins a judge model. You may set the judge `model` and a
`reasoning_effort` hint; everything else uses platform defaults.

## Deploy a semantic verifier

Deploying creates the verifier on first call and appends a new version on every
later call under the same name. A verifier name is unique per owner: lowercase
letters, digits, and hyphens, up to 128 characters.

The deploy payload is a single JSON object:

```json
{
  "name": "claims-cite-source",
  "description": "Every factual claim is backed by the provided source.",
  "criterion": "Return passed=true when every factual claim in the response is supported by the provided source text, otherwise passed=false.",
  "input_contract": "text",
  "input_fields": ["response", "source"],
  "few_shot_examples": [
    {
      "inputs": {"response": "...", "source": "..."},
      "passed": true,
      "reasoning": "Every claim traces to the source."
    },
    {
      "inputs": {"response": "...", "source": "..."},
      "passed": false,
      "reasoning": "The second sentence has no support in the source."
    }
  ],
  "model_settings": {"model": "anthropic/claude-sonnet-4-6", "reasoning_effort": "medium"}
}
```

**CLI.** The deploy command reads the JSON object from a file or from stdin
(stdin is preferred for generated agent output):

```sh
goodeye verifiers deploy ./claims-cite-source.json
# or pipe it in:
cat ./claims-cite-source.json | goodeye verifiers deploy -
```

On success it prints the `verifier_id`, the new `version`, and a
`version_token`. Persist the token: you need it for the next re-deploy.

**MCP tool.** `deploy_verifier`

**REST.**

```http
POST /v1/verifiers
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json
```

The response carries `{verifier_id, name, current_version, version,
version_token, status, input_contract, config_hash}`.

### Versioning and the version token

Each verifier carries a `version_token` for optimistic concurrency. The first
deploy of a name must omit `expected_version_token`. Every later deploy under
that name must include the latest token (from the previous deploy response,
`list`, or `show`). If the token does not match the current one, the deploy
returns a conflict (409) with the current token, so two callers cannot silently
clobber each other. A successful re-deploy appends a new version and rotates the
token.

Versions are immutable once written. To change a criterion or its calibration,
deploy a new version; old versions keep running for anyone who pinned them.

**Note:** Deploying a brand-new verifier whose `name` matches an active
platform-managed verifier is rejected with a conflict (409). Those names are
reserved (see [Platform-managed verifiers](#platform-managed-system-verifiers)).

## List, show, run, revoke, delete

### List

Lists the active (non-revoked) verifiers you own or can access through a
workflow grant. Platform-managed verifiers never appear here.

- CLI: `goodeye verifiers list` (add `--json`, `--table`, `--all`)
- MCP tool: `list_verifiers`
- REST: `GET /v1/verifiers`

### Show

Returns one verifier version in full: criterion, input contract, input fields,
calibration examples, judge config, and a `config_hash` for drift detection.
Defaults to the current version; pin one with `--version`. Requires owner or
tune access; anything you cannot tune returns 404.

- CLI: `goodeye verifiers show <id-or-name> [--version N]`
- MCP tool: `get_verifier`
- REST: `GET /v1/verifiers/{verifier_id}` (add `?version=N` to pin)

### Run

Runs the judge against your inputs and returns a pass or fail with the judge's
reasoning. The `inputs` keys must match the version's `input_fields` exactly;
supply `media_url` (a public HTTPS image URL) when the contract is `text_image`
or `image`.

```sh
goodeye verifiers run claims-cite-source \
  --inputs-json '{"response": "...", "source": "..."}'
```

- CLI: `goodeye verifiers run <id-or-name>` (`--inputs-json`, `--media-url`,
  `--version`, `--workflow-id`, `--run-id`, `--anonymous`, `--json`)
- MCP tool: `run_verifier`
- REST: `POST /v1/verifiers/{verifier_id}/runs`

```http
POST /v1/verifiers/<verifier_id>/runs
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{"inputs": {"response": "...", "source": "..."}, "version": 2}
```

A successful run returns `{verifier_run_id, anonymous_verifier_run_id,
verifier_id, version, status, passed, reasoning, duration_ms, created_at}` (one
of the two run-id fields is populated). The CLI exits 0 on any completed
judgment regardless of pass or fail: check the PASS/FAIL line or the JSON
`passed` field before you gate a downstream action on it.

**Error handling.** A caller-shape error (input keys do not match, wrong media
for the contract) returns 400 with no run row written. A judge runtime error
returns a row with `status="error"` and an `error_code` of `runtime_error`,
`verifier_unavailable`, or `timeout`; the CLI exits 1 and prints the code.

### Pin a version

Pin a specific version with `verifier_id@version` (for example,
`6f1c...@2`) wherever a workflow references a verifier, or with `--version` /
`?version=N` on a run. With no version, the run uses the current version.

### Revoke

Deactivates a verifier you own. It disappears from list, show, and run; existing
run rows are kept for audit. Revoke is irreversible: replace a revoked verifier
by deploying a fresh one under a new name.

- CLI: `goodeye verifiers revoke <id-or-name>` (`--yes` to skip the prompt)
- MCP tool: `revoke_verifier`
- REST: `DELETE /v1/verifiers/{verifier_id}`

### Delete (permanent)

Permanently and immediately erases a verifier you own: the verifier, all its
versions (criterion, calibration examples, input contracts), all run records,
and all access grants. There is no recovery path. Prefer revoke if you only want
to deactivate the verifier while keeping the audit trail.

A serving gate refuses deletion (409) when any live published template version
carries a snapshot that references the verifier. Unpublish the relevant template
version(s) first, then retry.

- CLI: `goodeye verifiers delete <id>` (UUID only; `--yes` to skip the prompt)
- MCP tool: `delete_verifier`
- REST: `DELETE /v1/verifiers/{verifier_id}/permanent`

**Note:** Revoke and delete are owner-only, and both accept your own verifiers
only. Pointing at someone else's verifier returns 404 (existence masking).

## How workflows reference verifiers

A workflow names the deployed verifiers it depends on by `verifier_id` or
`verifier_id@version`. At run time the agent invokes `run_verifier` with the
inputs the verifier's contract expects, then gates its next step on the returned
`passed`. Structural and functional checks stay inline in the workflow body;
semantic verifiers are referenced by ID rather than embedded, so a redeploy can
ship a sharper criterion without rewriting the workflow.

When you grant a workflow to another user or team, the semantic verifiers it
references can cascade with the grant so the grantee's agent can run them too.
See [Workflows](workflows.md) for grants, and [Templates](templates.md) for how
public templates freeze a verifier snapshot at publish time.

## Platform-managed (system) verifiers

Some verifiers are platform-managed. They are run-only: you invoke them through
a `system:<name>` alias (for example, `system:workflow-design-qa`), but their
criterion, calibration, and judge configuration are never exposed. They do not
appear in `list_verifiers`, cannot be fetched with `get_verifier`, and cannot be
revoked, deleted, deployed, or forked. Their names are reserved, so you cannot
deploy a new verifier that reuses one.

To run one, pass the alias as the verifier id:

```sh
goodeye verifiers run system:workflow-design-qa \
  --inputs-json '{"...": "..."}'
```

```http
POST /v1/verifiers/system:workflow-design-qa/runs
```

System verifiers run through the same metered path as your own, and the same
billing gates apply.

## Running a verifier anonymously

Most verifier operations require auth. The one exception is running a verifier
that a published template depends on. When a template version is live and its
snapshot pins a `(verifier_id, version)` pair, anonymous REST callers may run
that exact pair, so a template's published checks work for everyone who fetches
it. The judge still runs against the immutable deployed version, not the snapshot
copy.

```http
POST /v1/verifiers/<verifier_id>/runs
Content-Type: application/json

{"inputs": {"response": "...", "source": "..."}, "version": 3}
```

Anonymous spend draws on a small per-caller credit grant, the same ledger that
meters authenticated runs (there is no separate per-IP run-count cap). From the
CLI, pass `--anonymous` (which requires a verifier UUID or a `system:<name>`
alias, since names cannot be resolved without auth). Anonymous verifier
execution is REST-only; MCP always requires auth.

## Billing

Every semantic verifier run, yours or anonymous, draws on your credit balance.
If the balance is exhausted the run returns 402 `budget_exhausted`; a suspended
account returns 403 `account_suspended`. Anonymous runs are also subject to a
platform-wide daily limit on total anonymous usage, which returns 402
`anonymous_daily_cap` once reached (it resets at UTC midnight). Check granted,
used, and remaining
credit with `goodeye usage` (or `GET /v1/me/usage`). See
[Accounts and billing](accounts-and-billing.md) for tiers and grants.

## See also

- [Overview](overview.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [Image generators](image-generators.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
