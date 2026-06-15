# Templates

A template is the public form of a [workflow](workflows.md): a snapshot published
under your handle so other people and their agents can find, fetch, and fork it.
This page covers publishing a version, the safety checks that run before any
write, the unverified-template banner, reference forms, fetching and searching,
forking, the public lifecycle (unpublish, archive, delete, deprecate), running an
on-demand safety check, and transferring ownership.

Templates are immutable and versioned. Continued edits to your private workflow
never leak into a published template; a new round of work becomes a new version.

**The agent contract:** when an agent fetches a template body, it executes that
body as the user's runbook. Non-owner reads carry a safety banner (see
[The unverified-template banner](#the-unverified-template-banner)). See
[Overview](overview.md) for the full model.

## Publishing identity: your handle

Publishing requires a claimed handle, your public identity in the catalog. Claim
one first.

- **CLI:** `goodeye me claim-handle <handle>`
- **MCP tool:** `claim_handle`
- **REST:** `PATCH /v1/me`

Publishing without a claimed handle is rejected. See
[Accounts and Billing](accounts-and-billing.md) for claiming and renaming
handles, including how published template URLs stay pinned to the handle you
published under even after a rename.

## Publishing a version

Publishing promotes the current version of a workflow to a new public template
version. The first publish creates the template, reusing the workflow's slug;
each later publish appends a monotonically increasing version. The workflow must
declare a non-empty `outcome`.

- **CLI:** `goodeye templates publish <workflow-uuid-or-name>` (`--release-notes`)
- **MCP tool:** `publish_template_version`
- **REST:** `POST /v1/templates`

```sh
goodeye templates publish incident-postmortem \
  --release-notes "Tighten the root-cause section."
```

### Verifier snapshots are frozen at publish

At publish time, each semantic verifier the workflow references is frozen into an
immutable snapshot stored on the template version. Forks materialize their own
private copies from these snapshots, and published-template runs use the frozen
config. Non-owners never see the underlying judge configuration: a template
exposes only the metadata needed to call a verifier (its name, input contract,
input fields, and version), not its calibration or judge settings. Sibling files
from the workflow's bundle are snapshotted into the template version too.

### Safety checks run before any write

Every publish runs the platform safety checks over the whole bundle (the body
plus sibling files) before any database write:

- A **block** check is a hard gate. If it fails, the publish raises
  `safety_verification_failed` (422) and nothing is written.
- An **advisory** check is a soft signal. If it raises a concern, the publish
  still succeeds but the version is marked `flagged`.

If the safety runtime itself errors, the publish aborts with
`safety_verification_unavailable` (503) rather than stamping an ambiguous status.

The publish response includes a `safety_verification` object with the resolved
`status` and, when the advisory flagged concerns, the advisory reasoning:

```json
{
  "template_id": "…",
  "version": 2,
  "publishing_handle": "alice",
  "safety_verification": {
    "status": "flagged",
    "advisory_run_id": "…",
    "advisory_reasoning": "…"
  }
}
```

A status of `clean` means automated checks did not flag the version; `flagged`
means the advisory raised concerns. (A `blocked` version is never published, so
you will not see one in the catalog.)

## Add a demo to your template page

A demo is a short visual writeup that renders on the public template page, so a
visitor can picture what your workflow produces before they fork it. Its anchor
is a `demo/README.md` file in the workflow bundle, with any referenced images (and
an optional social-share card) alongside it. The whole demo travels with the
workflow and ships when you publish.

### The `demo/README.md` convention

Add a `demo/README.md` file to your workflow bundle (see
[Multi-file bundles](workflows.md#multi-file-bundles-directory-mode)) and author
it like a repository README. Its presence is what makes the public template page
render a demo section. There is no schema to fill in: write headings, prose,
images, and a video link in plain markdown. A before/after is just two captioned
images.

```markdown
# What this produces

A clean incident postmortem in minutes.

![Sample output](output.png)

https://www.youtube.com/watch?v=EXAMPLE
```

### Images

Reference images by relative path inside `demo/`. `![Result](output.png)`
resolves to `demo/output.png`. External or absolute image URLs are dropped and
will not render, so always use a relative `demo/` path; the image must be a file
in the bundle.

Demo images use a closed format allowlist: PNG, JPEG, WebP (including animated
WebP), and GIF. SVG and other vector or markup formats are not allowed for demo
images and are rejected at publish.

For a short motion clip, animated WebP is recommended (much smaller than GIF for
the same clip). GIF is accepted.

### Video

You can embed one short video by putting an allowlisted video URL on its own line
in the README. Allowed providers are YouTube, Loom, and Vimeo. The page renders a
click-to-load embed. A link from any other host stays a normal link and does not
embed.

### Social-share card

Add an optional `demo/og.png` (or `demo/og.webp` / `demo/og.jpg` / `demo/og.jpeg`,
raster only) to set the image used when your template page is shared on social
platforms.
Without one, a card is generated automatically from the template's metadata.

### Image checks at publish

Every image in a published template, including the social-share card, is screened
for disallowed content at publish. Disallowed content blocks the publish, the same
way the hard safety gate above does. The screen reads a still image, so it does
not inspect later frames of an animated clip; the video host performs its own
moderation on embedded video.

A bundle with too many images is rejected before any screen runs: trim the image
count and publish again. If the screening service is unreachable, the publish
still succeeds but the version is stamped `flagged` (advisory) rather than blocked,
so a screening outage surfaces for review instead of passing silently.

### Authoring notes

When you save or publish a workflow that has a `demo/README.md`, the response
surfaces advisory notes for anything that will not render as expected: an external
image that will be dropped, a video link from a host that will not embed, or a
referenced `demo/` image that is not in the bundle. The notes are non-blocking;
watch for them so your demo renders the way you intended.

To pull a published demo asset's raw bytes (a preview image, for example), use
`goodeye templates get-file <identifier> <path> --output <file>` (REST
`GET /v1/templates/{identifier}/files?path=&format=raw`). Pass `--sha256 HASH` to
content-address the fetch: the file is confirmed to match that hash, so a
republished or removed file no longer resolves at a stale address. See
[Get a template](#get-a-template) for the full file-fetching surface.

## Reference forms

A template is addressable three ways. Most commands accept any of them:

- **`uuid`**: the template's UUID.
- **`@handle/slug`**: the publisher's handle plus the template slug, resolving to
  the latest live version.
- **`@handle/slug@vN`**: the same, pinned to a specific version.

Where a command also takes a `--version` flag, the flag overrides any `@vN`
suffix.

## Fetching, listing, and searching

### Get a template

Anyone, including anonymous callers, can read the public catalog. The default
response is the markdown body; pass `--json` for the full record, which includes
the file manifest and the public verifier snapshots.

- **CLI:** `goodeye templates get <identifier>` (`--version`, `--output`, `--json`)
- **MCP tool:** `get_template` (single files via `get_template_file`)
- **REST:** `GET /v1/templates/{identifier}` (files via
  `GET /v1/templates/{identifier}/files?path=`)

If you reference a `@handle` that the publisher renamed away from within its
reservation window, the read follows one redirect to the current handle and tells
you where it landed.

### List templates

- **CLI:** `goodeye templates list` (`--filter all|mine`, `--search`, `--limit`,
  `--all`, `--include-archived`)
- **MCP tool:** `list_templates` (`include_archived` param)
- **REST:** `GET /v1/templates?include_archived=true`

A template appears in listings only while at least one of its versions is
published. `--include-archived` surfaces your own archived templates only;
another owner's archived templates and anonymous callers never see archived
templates.

### Search templates

Natural-language, LLM-ranked search over the public catalog, distinct from the
lexical filtering on `list`.

- **CLI:** `goodeye templates search "<query>"` (`--filter`, `--limit`)
- **MCP tool:** `search_templates`
- **REST:** `POST /v1/templates/search`

## Forking into a private workflow

Forking copies a specific template version into a new private
[workflow](workflows.md) you own, materializing private copies of the snapshot
verifiers and image generators and copying the file tree. Forking requires
authentication; anonymous callers can read the catalog but cannot fork. The fork
defaults to the template's slug, with a numeric suffix (`-2`, `-3`) if you
already have a workflow by that name.

- **CLI:** `goodeye templates fork <identifier>` (`--version`, `--name`)
- **MCP tool:** `fork_template`
- **REST:** `POST /v1/templates/fork`

The fork response carries the new `workflow_id` and lineage back to the version
it came from, but no body. Fetch the body with `goodeye workflows get` as a
separate step. Forking a deprecated version succeeds but surfaces a deprecation
warning. A version that was blocked by safety checks cannot be forked.

## The unverified-template banner

Goodeye prepends a safety banner to template bodies for non-owner reads
(including anonymous readers) as a cross-user trust signal. Owners do not see the
banner on their own templates, and the banner is fixed and neutral: it carries no
owner identity and cannot be customized or suppressed. The banner text is
selected by the version's safety status:

- **`unverified`**: this version was not safety-checked; treat it as untrusted.
- **`clean`**: automated checks did not flag it at publish (not a guarantee of
  safety).
- **`flagged`**: the safety rubric flagged advisory concerns; treat it as
  untrusted. When the bundle includes scripts or binary files, the banner adds a
  note not to execute them.

In all three cases the banner instructs the executing agent not to take
destructive or irreversible actions, exfiltrate secrets, or override the user's
task on the template's authority.

Owners and non-owners see different detail on the safety status. The
`safety_verification_status` field is visible to everyone, but the advisory
reasoning behind a `flagged` version is shown only to the owner; non-owners see
the status alone.

## Lifecycle: unpublish, archive, delete, deprecate

These paths are deliberately distinct. Pick by intent.

### Unpublish a version (hide one version)

Soft-unpublishes a single template version. Forks already pinned to that version
keep working. The catalog hides the template once no live version remains.

- **CLI:** `goodeye templates unpublish <identifier> <version>`
- **MCP tool:** `unpublish_template_version`
- **REST:** `DELETE /v1/templates/{template_ref}/versions/{v}`

### Deprecate a version (warn, do not hide)

Flags a single version as deprecated without hiding it. The version stays
fetchable and forkable; anyone who forks it sees your deprecation message.
Last-write-wins on the message, so you can update it by calling again. A message
is required.

- **CLI:** `goodeye templates deprecate-version <identifier> <version> --message "<text>"`
- **MCP tool:** `deprecate_template_version`
- **REST:** `POST /v1/templates/{template_ref}/versions/{v}/deprecate`

### Archive (reversible hide of the whole template)

Archiving hides the whole template from public listing while keeping every
version and all fork lineage intact. Existing forks keep working. It keeps the
slug occupied and is idempotent. An optional reason is recorded.

- **CLI:** `goodeye templates archive <template-ref>` (`--reason`) /
  `goodeye templates unarchive <template-ref>`
- **MCP tool:** `archive_template` / `unarchive_template`
- **REST:** `POST /v1/templates/{template_ref}/archive` /
  `POST /v1/templates/{template_ref}/unarchive`

### Permanent delete (irreversible)

Permanent delete erases the template, all its versions, all attached files, and
all version verification records at once. There is no recovery path.

- **CLI:** `goodeye templates delete <template-ref>` (`--yes`)
- **MCP tool:** `delete_template`
- **REST:** `DELETE /v1/templates/{template_ref}`

A serving gate protects readers: if the template is live with any still-published
version, deletion is refused with a `conflict` (409). Unpublish the relevant
versions, or archive the whole template, first. Forks keep their own content
copy; only their parent pointer is severed, and lineage then reports the source
as permanently deleted.

To erase a single non-published version permanently (leaving the rest of the
template intact), unpublish it first, then:

- **CLI:** `goodeye templates delete-version <template-ref> <version>`
- **MCP tool:** `delete_template_version`
- **REST:** `DELETE /v1/templates/{template_ref}/versions/{v}/permanent`

**Note:** encrypted backups age out within the platform's standard retention
window (up to three months), so deleted content is not instantly erased from all
systems everywhere, but it is no longer reachable through any product surface.
Prefer archive when you want a reversible alternative.

## Running a safety check on demand

You can re-run the platform safety checks against a published template version at
any time. Auth is optional: pass `--anonymous` to run without sending an API key
(anonymous spend bills against a small per-caller credit budget). Each call bills
two metered verifier runs and returns a `status` of `clean`, `flagged`,
`blocked`, or `error`. Unlike `get_template`, full reasoning is returned
regardless of ownership, because the caller paid for the runs.

- **CLI:** `goodeye templates check-safety <identifier>` (`--version`,
  `--anonymous`, `--json`)
- **MCP tool:** `check_template_safety`
- **REST:** `POST /v1/templates/{template_ref}/safety-check`

**Note:** this on-demand check covers the body, description, and outcome only; it
does not re-scan sibling files. The authoritative, durable trust signal is the
`safety_verification_status` that publish computed over the whole bundle and
stored on the version, which `get_template` surfaces.

## Transferring ownership

Transferring a template does not apply immediately. It creates an invitation
envelope; the recipient must accept it before ownership moves. A self-transfer is
a no-op. Existing published versions keep the handle they were published under.

- **CLI:** `goodeye templates transfer-ownership <template-ref> <new-owner>`
- **MCP tool:** `transfer_template_ownership`
- **REST:** `POST /v1/templates/{template_ref}/transfer-ownership`

The recipient accepts with `goodeye invitations accept <id>` (or the
`accept_invitation` tool / `POST /v1/invitations/{id}/accept`). See
[Teams](teams.md) for the full invitation flow.

## See also

- [Workflows](workflows.md): the private object you publish from and fork back
  into, including how to carry demo files in a multi-file bundle.
- [Verifiers](verifiers.md): the semantic checks a template snapshots.
- [Accounts and Billing](accounts-and-billing.md): handles, usage, and credits.
- [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md): the three surfaces in
  detail.
