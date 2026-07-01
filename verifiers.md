# Verifiers

A verifier is a check a workflow runs on agent output to confirm the work hit a
measurable result. This page covers the three verifier types, how to deploy and
version your own semantic verifiers, and how workflows reference them.

A verifier is always outcome-specific. It scores one concrete property of the
output ("did the response cite a source for every claim?", "is the JSON valid
against this schema?"). It is never a holistic "is this good overall?" check. If
you cannot say what a pass and a fail look like in one sentence, the check is not
specific enough yet.

For where verifiers fit in the larger picture, see [Overview](overview.md). For
the runbooks that invoke them, see [Workflows](workflows.md).

## The three verifier types

A single workflow usually combines all three. They differ in what they check and
where they run.

```diagram-verifier-types
head: Verifier | a check the workflow runs on agent output, pass or fail with reasoning
Structural | Format and schema | required fields, the shape parses | inline, free
Functional | Tests and bounds | tests, ranges, regex, hashes | inline, free
* Semantic | Interpretive judgment | tone, factuality, image quality | judge runtime
```

### Structural

Format and schema checks: required fields are present, the output parses, the
shape matches. These are deterministic and live inline in the workflow body as
plain code or assertions. There is nothing to deploy; the workflow runs them
itself.

Use a structural check when a pass or fail is decidable by inspecting the shape of
the output (valid JSON, all required keys present, the right number of rows).

### Functional

Tests, bounds, and computed checks: unit tests pass, a number lands inside a
range, a regular expression matches, a hash compares equal. Like structural
checks, functional checks are deterministic and live inline in the workflow body.

Use a functional check when a pass or fail is decidable by running code against
the output (a test suite, a numeric tolerance, a checksum).

### Semantic

Interpretive judgments that need reading and reasoning, not just parsing: tone,
factuality against a source, pedagogical quality, whether an image matches a
brief. A semantic verifier is one judgment ("does this output satisfy this
criterion?") evaluated by an LLM judge, calibrated with labeled examples.

Unlike the other two types, a semantic verifier is a stored, versioned object that
you deploy, run, and reference by ID. The rest of this page is about semantic
verifiers.

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
  - `image`: a single image only, no text fields (the chart-design check in the
    high-signal chart workflow is an `image` verifier).

| Contract | input_fields | Image required | Example |
|---|---|---|---|
| `text` | One or more named text fields | No | Claim cites its source |
| `text_image` | Named text fields plus one image | Yes | Caption matches the image |
| `image` | None | Yes | Chart-design check |

The contract you choose determines what `run_verifier` accepts. For `text` and
`text_image` you declare `input_fields` (the named text inputs); for `image` you
leave them empty. A run must supply exactly those fields, no more and no fewer, and
must include an image URL when the contract calls for one.

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
  "model_settings": {"model": "openai/gpt-5.4-mini", "reasoning_effort": "medium"}
}
```

**CLI.** The deploy command reads the JSON object from a file or from stdin (stdin
is preferred for generated agent output):

```sh
goodeye verifiers deploy ./claims-cite-source.json
# or pipe it in:
cat ./claims-cite-source.json | goodeye verifiers deploy -
```

On success it prints the `verifier_id`, the new `version`, and a `version_token`.
Persist the token: you need it for the next re-deploy.

**MCP tool.** `deploy_verifier`

**REST.** `POST /v1/verifiers` (see [REST API](rest-api.md) for the request and
response shape).

### Versioning and the version token

Versions are immutable once written: to change a criterion or its calibration,
deploy a new version, and old versions keep running for anyone who pinned them.
Each deploy is guarded by a `version_token`, the same guard used when saving a
workflow (see [Updating safely](workflows.md#updating-safely)):
the first deploy of a name omits the token, every later deploy includes the latest
one, and a mismatch returns a `conflict` (409) so two callers cannot clobber each
other.

**Note:** Deploying a brand-new verifier whose `name` matches an active
platform-managed verifier is rejected with a conflict (409). Those names are
reserved (see [Platform-managed verifiers](#platform-managed-system-verifiers)).

## List, show, run, revoke, delete

### List

Lists the active (non-revoked) verifiers you own or can access through a workflow
grant. Platform-managed verifiers never appear here.

- **CLI:** `goodeye verifiers list` (add `--json`, `--table`, `--all`)
- **MCP tool:** `list_verifiers`
- **REST:** `GET /v1/verifiers`

### Show

Returns one verifier version in full: criterion, input contract, input fields,
calibration examples, and judge config. Defaults to the current version; pin one
with `--version`. Anyone who can reach the verifier can read it in full: the owner,
and anyone you grant the workflow to, at any role (including `view`). Deploying a
new version needs `edit` or `admin` on the workflow. Anything you cannot reach
returns 404.

- **CLI:** `goodeye verifiers show <id-or-name> [--version N]`
- **MCP tool:** `get_verifier`
- **REST:** `GET /v1/verifiers/{verifier_id}` (add `?version=N` to pin)

### Run

Runs the judge against your inputs and returns a pass or fail with the judge's
reasoning. The `inputs` keys must match the version's `input_fields` exactly;
supply `media_url` (a public HTTPS image URL) when the contract is `text_image` or
`image`.

```sh
goodeye verifiers run claims-cite-source \
  --inputs-json '{"response": "...", "source": "..."}'
```

- **CLI:** `goodeye verifiers run <id-or-name>` (`--inputs-json`, `--media-url`,
  `--version`, `--workflow-id`, `--run-id`, `--anonymous`, `--json`)
- **MCP tool:** `run_verifier`
- **REST:** `POST /v1/verifiers/{verifier_id}/runs`

A successful run returns the verdict (`passed`), the judge's `reasoning`, and run
metadata. The CLI exits 0 on any completed judgment regardless of pass or fail:
check the PASS/FAIL line or the JSON `passed` field before you gate a downstream
action on it.

**Error handling.** A caller-shape error (input keys do not match, wrong media for
the contract) returns 400 with no run row written. A judge runtime error returns a
row with `status="error"` and an `error_code` of `runtime_error`,
`verifier_unavailable`, or `timeout`; the CLI exits 1 and prints the code.

### Pin a version

Pin a specific version with `verifier_id@version` wherever a workflow references a
verifier, or with `--version` / `?version=N` on a run. With no version, the run
uses the current version.

### Revoke

Deactivates a verifier you own. It disappears from list, show, and run; existing
run rows are kept for audit. Revoke is irreversible: replace a revoked verifier by
deploying a fresh one under a new name.

- **CLI:** `goodeye verifiers revoke <id-or-name>` (`--yes` to skip the prompt)
- **MCP tool:** `revoke_verifier`
- **REST:** `DELETE /v1/verifiers/{verifier_id}`

### Delete (permanent)

Permanently and immediately erases a verifier you own: the verifier, all its
versions, all run records, and all access grants. There is no recovery path. Prefer
revoke if you only want to deactivate the verifier while keeping the audit trail.

Deletion is refused (409) while any live published template version still
references the verifier. Unpublish the relevant template version(s) first, then
retry.

- **CLI:** `goodeye verifiers delete <id>` (UUID only; `--yes` to skip the prompt)
- **MCP tool:** `delete_verifier`
- **REST:** `DELETE /v1/verifiers/{verifier_id}/permanent`

**Note:** Revoke and delete are owner-only and accept your own verifiers only.
Pointing at someone else's verifier returns 404.

## How workflows reference verifiers

A workflow names the deployed verifiers it depends on by `verifier_id` or
`verifier_id@version`. At run time the agent invokes `run_verifier` with the inputs
the verifier's contract expects, then gates its next step on the returned `passed`.
Structural and functional checks stay inline in the workflow body; semantic
verifiers are referenced by ID rather than embedded, so a redeploy can ship a
sharper criterion without rewriting the workflow.

When you grant a workflow to another user or team, the semantic verifiers it
references cascade with the grant: the grantee's agent can run them, and because a
verifier you can reach is fully readable (criterion and calibration examples
included), collaborators can see and improve the grader instead of tuning against a
black box. Writing stays gated: deploying a new version needs `edit` access, and
revoking, deleting, or rewiring which verifiers a workflow references stays with the
owner. See [Workflows](workflows.md) for grants, and [Templates](templates.md) for
how public templates publish verifier definitions.

## Platform-managed (system) verifiers

Some verifiers are platform-managed. They are run-only: you invoke them through a
`system:<name>` alias (for example, `system:workflow-design-qa`), but their
criterion, calibration, and judge configuration are never exposed. They do not
appear in `list_verifiers`, cannot be fetched with `get_verifier`, and cannot be
revoked, deleted, deployed, or forked. Their names are reserved, so you cannot
deploy a new verifier that reuses one.

To run one, pass the alias as the verifier id:

```sh
goodeye verifiers run system:workflow-design-qa \
  --inputs-json '{"...": "..."}'
```

System verifiers run through the same metered path as your own, and the same
billing gates apply.

## Running a verifier anonymously

Most verifier operations require auth. The one exception is running a verifier that
a published template depends on: when a template version is live and its snapshot
pins a `(verifier_id, version)` pair, anonymous REST callers may run that exact
pair, so a template's published checks work for everyone who fetches it. The judge
still runs against the immutable deployed version, not the snapshot copy.

From the CLI, pass `--anonymous` (with a verifier UUID or a `system:<name>` alias,
since names cannot be resolved without auth). Anonymous execution is REST-only; MCP
always requires auth. Anonymous spend draws on a small per-caller credit grant, the
same balance that meters authenticated runs.

## Billing

Every semantic verifier run, yours or anonymous, draws on your credit balance. An
exhausted balance returns 402 `budget_exhausted`, a suspended account returns 403
`account_suspended`, and anonymous runs can also hit 402 `anonymous_daily_cap`.
Check granted, used, and remaining credit with `goodeye usage` (or
`GET /v1/me/usage`). See [Accounts and Billing](accounts-and-billing.md) for tiers,
grants, and the full error reference.

## See also

- [Overview](overview.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [Image generators](image-generators.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and Billing](accounts-and-billing.md)
