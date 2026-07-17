# Image generators

An image generator is a deployed, owner-scoped image-generation capability that a
skill can call. You deploy one once with a fixed model and default parameters,
then reference it by ID from a skill so the agent can produce images that meet the
standard you defined.

This page covers deploying and managing your own generators, the three ways to
call image generation, and how anonymous and billing behavior works. For where
generators fit alongside everything else, see [Overview](overview.md); for the
runbooks that call them, see [Skills](skills.md).

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

A generator name is unique per owner: lowercase letters, digits, and hyphens,
up to 128 characters. The first deploy creates it; later deploys under the same
name append versions.

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

The response carries the `generator_id`, the new `version`, and a
`version_token`; see [REST API](rest-api.md) for the full field list.

### Versioning and the version token

Re-deploying a generator uses the same version-token guard as skills: every
deploy after the first must pass the latest `version_token`, and a mismatch is
rejected with a conflict (409) so two callers cannot clobber each other. See
[Skills](skills.md) for the concept.

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

Returns one generator version in full: model, contract, and default parameters.
Defaults to the current version; pin one with `--version`. Owner-only; a
generator you do not own returns 404.

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

A serving gate refuses deletion (409) when a live published template version
still references the generator. Unpublish the relevant template version(s)
first, then retry.

- CLI: `goodeye image-generators delete <id>` (`--yes` to skip the prompt)
- MCP tool: `delete_image_generator`
- REST: `DELETE /v1/image-generators/{generator_id}/permanent`

**Note:** Revoke and delete are owner-only. Pointing at someone else's generator
returns 404.

## Generate an image

Generation takes a `prompt` and returns one or more image URLs. For an
`image_to_image` generator, supply `reference_image_url` (a public HTTP or HTTPS
URL); for `text_to_image`, supplying a reference image is rejected. You can
request 1 to 4 images per call (`num_images`), and each image is billed
separately.

- CLI: `goodeye image-generators generate` (`--prompt`, `--generator`,
  `--model`, `--reference-image-url`, `--num-images`, `--seed`, `--visibility`,
  `--params-json`, `--version`, `--anonymous`, `--skill-id`, `--run-id`,
  `--json`)
- MCP tool: `generate_image`
- REST: `POST /v1/image-generators/{generator_id}/runs`

The CLI prints image URLs to stdout, one per line (so the result pipes cleanly),
with cost and run metadata on stderr or in `--json` mode. A successful call
returns the run id, the generated `image_urls`, per-image cost, and a
`hosted_images` list; see [REST API](rest-api.md) for the full field list. Each
output is also auto-hosted on Goodeye, so its hosted URL stays valid after the
provider session ends; see [Image hosting](images.md).

### Image visibility

Each generated image is hosted as a **public** image by default; set
`visibility` to `private` (CLI `--visibility private`, MCP/REST
`visibility: "private"`) to keep it private with a shareable view link instead.
Anonymous generations are always public. See [Image hosting](images.md) for how
hosting, visibility, and view links work.

### Call modes

There are three ways to choose what generates the image:

| Call mode | How you reference it | From a published template | Anonymous |
|---|---|---|---|
| System tier | `system:<tier>` (e.g. `system:image-standard`) | Yes | Yes |
| Deployed generator | `<uuid>` or `<uuid>@<version>` | Yes, if the template references it | Yes, same case |
| Ephemeral model | `--model <slug>` (or `model` in the body) | No | No |

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
   skill uses to call a generator you own.

   ```sh
   goodeye image-generators generate \
     --generator 6f1c0a2e-...@2 \
     --prompt "A product hero shot lit from the left, white background"
   ```

3. **An ephemeral `model=` slug.** Pass `--model <slug>` for an authenticated
   one-off against a concrete model identifier, with no deployed generator. The
   contract is inferred from whether you supply a reference image.

   ```sh
   goodeye image-generators generate \
     --model provider-org/model-name \
     --prompt "A flat-lay product shot on a neutral background"
   ```

`--generator` and `--model` are mutually exclusive: supply one or the other.

On REST, the path segment is the generator reference (a UUID, `uuid@version`, or
`system:<tier>`). To use the one-off model path, set `model` in the body and use
any placeholder path segment; when `model` is set the path id is ignored.

```http
POST /v1/image-generators/system:image-standard/runs
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"prompt": "A product hero shot, soft studio lighting", "num_images": 1}
```

### Safety checker

The provider safety checker is forced on for system tiers, anonymous callers,
and any image hosted publicly (the default): those paths serve untrusted,
unauthenticated, or world-readable output and cannot opt out of safety
filtering. Only your own deployed generators called with auth and
`visibility=private` keep control of the checker, where your generator's default
parameters apply.

### The prompt is never stored

A generation prompt is never persisted. Only a one-way hash of the prompt is kept
on the run record for correlation, and the provider credential is never logged or
returned. Generated image URLs are returned to you and recorded on the run row.

## Anonymous generation and billing

Anonymous image generation is available over REST only; the MCP surface always
requires auth. An anonymous caller may invoke a system tier or a deployed
generator that a live published template references (the path a published
template uses to generate images for anyone who fetches it); the ephemeral
`model=` slug path is never available anonymously. From the CLI, pass
`--anonymous`.

Every generation, yours or anonymous, draws on your credit balance, and each
image in a multi-image call is billed separately; anonymous spend uses the same
credit balance that meters authenticated runs. A spent balance, a suspended
account, or the anonymous limit blocks the call before any image is produced;
see [Accounts and billing](accounts-and-billing.md) for tiers, grants, and the
exact errors. Provider and timeout failures are different: they return a
completed call with `status="error"` and an `error_code` of `provider_error`,
`runtime_error`, or `timeout` (the CLI exits 1). Check remaining credit with
`goodeye usage` (or `GET /v1/me/usage`).

## See also

- [Overview](overview.md)
- [Skills](skills.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
