# Image generators

An image generator is a deployed, owner-scoped image-generation capability that a
workflow can call. You deploy one once with a fixed model and default parameters,
then reference it by ID from a workflow so the agent can produce images on the
outcome you defined.

This page covers deploying and managing your own generators, the three ways to
call image generation, and how anonymous and billing behavior works. For where
generators fit alongside the rest of the registry, see [Overview](overview.md);
for the runbooks that call them, see [Workflows](workflows.md).

## What a generator is

A generator pins three things at deploy time:

- A **model**: the provider image endpoint that does the generation.
- A **generation contract**: the input shape, either `text_to_image` (a prompt
  only) or `image_to_image` (a prompt plus a reference image URL).
- **Default parameters**: per-generator defaults (such as image size) that
  per-call overrides merge on top of at run time.

Like verifiers, generators are private and versioned: each redeploy under the
same name appends a new version, and the model and parameters of a given version
are immutable.

## Deploy a generator

Deploying creates the generator on first call and appends a new version on every
later call under the same name. A generator name is unique per owner: lowercase
letters, digits, and hyphens, up to 128 characters.

**Pricing is validated at deploy time.** The model you pass must resolve to a
known image endpoint with an authoritative per-image price. A model that cannot
be priced is rejected (400) before any record is written, so a deployed
generator always has a price the meter can compute against at run time.

**CLI.**

```sh
goodeye image-generators deploy \
  --name product-hero \
  --description "Square hero shots for product pages." \
  --model provider-org/model-name \
  --generation-contract text_to_image \
  --params-json '{"image_size": "square_hd"}'
```

On success it prints the `generator_id`, the new `version`, and a
`version_token`. Persist the token: you need it for the next re-deploy.

**MCP tool.** `deploy_image_generator`

**REST.**

```http
POST /v1/image-generators
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "name": "product-hero",
  "description": "Square hero shots for product pages.",
  "model": "provider-org/model-name",
  "generation_contract": "text_to_image",
  "default_params": {"image_size": "square_hd"}
}
```

The response carries `{generator_id, name, current_version, version,
version_token, status, provider, model, generation_contract, config_hash}`.

### Versioning and the version token

The first deploy of a name must omit `expected_version_token`. Every later deploy
under that name must include the latest token (from the previous deploy response,
`list`, or `show`). A token mismatch returns a conflict (409) with the current
token, so two callers cannot clobber each other. A successful re-deploy appends a
new version and rotates the token.

**Note:** Deploying a new generator whose `name` matches an active
platform-managed generator is rejected with a conflict (409). System tier names
are reserved (see [System tiers](#call-modes)).

## List, show, revoke, delete

### List

Lists the active (non-revoked) generators you own. Platform-managed generators
never appear here.

- CLI: `goodeye image-generators list` (add `--json`, `--table`, `--all`)
- MCP tool: `list_image_generators`
- REST: `GET /v1/image-generators`

### Show

Returns one generator version in full: model, contract, default parameters, and
a `config_hash` for drift detection. Defaults to the current version; pin one
with `--version`. Owner-only; a generator you do not own returns 404.

- CLI: `goodeye image-generators show <id> [--version N]`
- MCP tool: `get_image_generator`
- REST: `GET /v1/image-generators/{generator_id}` (add `?version=N` to pin)

### Revoke

Deactivates a generator you own. It disappears from list, show, and generate;
existing run rows are kept for audit. Revoke is irreversible: replace a revoked
generator by deploying a fresh one under a new name.

- CLI: `goodeye image-generators revoke <id>` (`--yes` to skip the prompt)
- MCP tool: `revoke_image_generator`
- REST: `DELETE /v1/image-generators/{generator_id}`

### Delete (permanent)

Permanently and immediately erases a generator you own: the generator, all its
versions, and all run records. There is no recovery path. Prefer revoke if you
only want to deactivate it while keeping the audit trail.

A serving gate refuses deletion (409) when any live published template version
carries a snapshot that references the generator. Unpublish the relevant template
version(s) first, then retry.

- CLI: `goodeye image-generators delete <id>` (`--yes` to skip the prompt)
- MCP tool: `delete_image_generator`
- REST: `DELETE /v1/image-generators/{generator_id}/permanent`

**Note:** Revoke and delete are owner-only. Pointing at someone else's generator
returns 404 (existence masking).

## Generate an image

Generation takes a `prompt` and returns one or more image URLs. For an
`image_to_image` generator, supply `reference_image_url` (a public HTTP or HTTPS
URL); for `text_to_image`, supplying a reference image is rejected. You can
request 1 to 4 images per call (`num_images`), and each image is billed
separately.

- CLI: `goodeye image-generators generate` (`--prompt`, `--generator`,
  `--model`, `--reference-image-url`, `--num-images`, `--seed`, `--params-json`,
  `--version`, `--anonymous`, `--workflow-id`, `--run-id`, `--json`)
- MCP tool: `generate_image`
- REST: `POST /v1/image-generators/{generator_id}/runs`

The CLI prints image URLs to stdout, one per line (so the result pipes cleanly),
with cost and run metadata on stderr or in `--json` mode. A successful call
returns `{run_id, model_tier_or_model, image_url, image_urls, width, height,
num_images, cost_usd, duration_ms, status, created_at, error_code,
error_message}`.

### Call modes

There are three ways to choose what generates the image:

1. **A system tier.** Pass a `system:<tier>` reference (for example,
   `system:image-standard`) to use a platform-managed quality tier. Tiers are
   run-only: they are not listed, fetched, revoked, or deployed, and a system
   tier always resolves to its newest version (auto-upgrade). With no
   `--generator` and no `--model`, generation defaults to the standard tier.

   ```sh
   goodeye image-generators generate \
     --generator system:image-standard \
     --prompt "A minimalist product hero shot on a white background"
   ```

2. **A deployed generator by UUID.** Pass `--generator <uuid>` (or
   `--generator <uuid>@<version>` to pin a version) to use one of your deployed
   generators with its configured model and defaults. This is the mode a
   workflow uses to call a generator you own.

   ```sh
   goodeye image-generators generate \
     --generator 6f1c0a2e-...@2 \
     --prompt "A minimalist product hero shot on a white background"
   ```

3. **An ephemeral `model=` slug.** Pass `--model <slug>` for an authenticated
   one-off against a concrete model identifier, with no deployed generator. The
   contract is inferred from whether you supply a reference image. This mode is
   not usable from published templates and not available to anonymous callers.

   ```sh
   goodeye image-generators generate \
     --model provider-org/model-name \
     --prompt "A minimalist product hero shot on a white background"
   ```

`--generator` and `--model` are mutually exclusive: supply one or the other.

On REST, the path segment is the generator reference (a UUID, `uuid@version`, or
`system:<tier>`). To use the one-off model path, set `model` in the body and use
any placeholder path segment; when `model` is set the path id is ignored.

```http
POST /v1/image-generators/system:image-standard/runs
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"prompt": "A minimalist product hero shot on a white background", "num_images": 1}
```

### Safety checker

The provider safety checker is forced on for system tiers and for anonymous
callers: those paths serve untrusted or unauthenticated input and cannot opt out
of safety filtering. For your own deployed generators called with auth, your
generator's default parameters apply.

### The prompt is never stored

A generation prompt is never persisted. Only a one-way hash of the prompt is kept
on the run record for correlation, and the provider credential is never logged or
returned. Generated image URLs are returned to you and recorded on the run row.

## Anonymous generation

Anonymous image generation is available over REST only; the MCP surface always
requires auth. An anonymous caller may invoke a system tier, or a deployed
generator UUID whose `(generator_id, generator_version)` appears in a live
published template snapshot. This is the path a published template uses to
generate images for anyone who fetches it. The ephemeral `model=` slug path is
never available anonymously.

From the CLI, pass `--anonymous` (which requires a system tier or a generator
UUID that appears in a published template). Anonymous spend draws on a small
per-caller credit grant, the same ledger that meters authenticated runs.

## Billing and the budget gate

Every generation, yours or anonymous, draws on your credit balance, and each
image in a multi-image call is billed separately. Billing-gate errors propagate
before any image is produced:

- 402 `budget_exhausted` when the credit balance is spent.
- 402 `anonymous_daily_cap` when the shared daily limit on anonymous usage is reached (anonymous callers only; resets at UTC midnight).
- 403 `account_suspended` when the account is suspended.

Provider and timeout errors are different: they return a completed call with
`status="error"` and an `error_code` of `provider_error`, `runtime_error`, or
`timeout` (the CLI exits 1 in that case). Check granted, used, and remaining
credit with `goodeye usage` (or `GET /v1/me/usage`). See
[Accounts and billing](accounts-and-billing.md) for tiers and grants.

## See also

- [Overview](overview.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
