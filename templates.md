# Templates

A template is the public form of a [workflow](workflows.md): a snapshot published
under your handle so other people and their agents can find, fetch, and fork it.
Templates are immutable and versioned, so continued edits to your private workflow
never leak into a published template; a new round of work becomes a new version.

```diagram-template-lifecycle
Private workflow | your editable runbook
Publish a version | an immutable, versioned snapshot
* Public template | anyone, or any agent, can find and run it
Fork | a saveable private copy, with lineage back
```

When an agent fetches a template body, it executes that body as the user's
runbook. Non-owner reads carry a safety banner (see
[The unverified-template banner](#the-unverified-template-banner)). See
[Overview](overview.md) for the full model.

## Publishing identity: your handle

Publishing requires a claimed handle, your public identity in the catalog. Claim
one first.

- **CLI:** `goodeye me claim-handle <handle>`
- **MCP tool:** `claim_handle`
- **REST:** `PATCH /v1/me`

Publishing without a claimed handle is rejected. See
[Accounts and Billing](accounts-and-billing.md) for claiming and renaming handles,
including how published template URLs stay pinned to the handle you published under
even after a rename.

## Publishing a version

Publishing promotes the current version of a workflow to a new public template
version. The first publish creates the template, reusing the workflow's slug; each
later publish adds the next version. The workflow must declare a non-empty
`outcome`.

- **CLI:** `goodeye templates publish <workflow-id-or-name>` (`--release-notes`)
- **MCP tool:** `publish_template_version`
- **REST:** `POST /v1/templates`

```sh
goodeye templates publish high-signal-chart \
  --release-notes "Tighten the takeaway annotation."
```

### Verifier snapshots are frozen at publish

At publish time, each semantic verifier the workflow references is frozen into an
immutable snapshot on the template version, and sibling files from the workflow's
bundle are snapshotted too. Forks materialize their own private copies from these
snapshots, and published-template runs use the frozen config. A published
template's verifier definitions are public: every reader, including anonymous
catalog readers, sees each verifier's full definition (criterion, calibration
examples, and judge config), so a reader can understand exactly how the workflow is
graded before forking it. Because the definitions go public, do not put secrets or
private data in a verifier you publish (the safety check below enforces this).

### Safety checks run before any write

Every publish runs the platform safety checks over the whole bundle (the body plus
sibling files) before any write:

- A **block** check is a hard gate. If it fails, the publish raises
  `safety_verification_failed` (422) and nothing is written.
- An **advisory** check is a soft signal. If it raises a concern, the publish still
  succeeds but the version is marked `flagged`.

The same checks cover the verifier definitions this publish makes public: a
high-confidence secret (an API key, credential, or token) embedded in a verifier's
criterion or calibration hard-blocks the publish, and possible private data flags
the version. When the template references verifiers, the publish response includes a
`verifier_exposure_notice` restating that the definitions are now public. If the
safety check itself cannot complete, the publish aborts with
`safety_verification_unavailable` (503) rather than stamping an ambiguous status.

The publish response includes a `safety_verification` object with the resolved
`status` and, when the advisory flagged concerns, its reasoning:

```json
{
  "template_id": "…",
  "version": 2,
  "publishing_handle": "alice",
  "safety_verification": {
    "status": "flagged",
    "advisory_reasoning": "…"
  }
}
```

A status of `clean` means automated checks did not flag the version; `flagged` means
the advisory raised concerns. A `blocked` version is never published, so you will
not see one in the catalog.

## Reference forms

A template is addressable three ways, and most commands accept any of them:

- **`uuid`**: the template's UUID.
- **`@handle/slug`**: the publisher's handle plus the template slug, resolving to
  the latest live version.
- **`@handle/slug@vN`**: the same, pinned to a specific version.

Where a command also takes a `--version` flag, the flag overrides any `@vN` suffix.

## Fetching, listing, and searching

### Get a template

Anyone, including anonymous callers, can read the public catalog. The default
response is the markdown body; pass `--json` for the full record, which includes the
file manifest and the verifier snapshots in full (each verifier's criterion,
calibration examples, and judge config).

- **CLI:** `goodeye templates get <identifier>` (`--version`, `--output`, `--json`)
- **MCP tool:** `get_template` (single files via `get_template_file`)
- **REST:** `GET /v1/templates/{identifier}` (files via
  `GET /v1/templates/{identifier}/files?path=`)

To pull a single published file's raw bytes (a demo image, say), use
`goodeye templates get-file <identifier> <path> --output <file>`
(`GET /v1/templates/{identifier}/files?path=&format=raw`); pass `--sha256 HASH` to
confirm the bytes match a known hash.

If you reference a `@handle` that the publisher renamed away from within its
reservation window, the read follows one redirect to the current handle and tells
you where it landed.

### List templates

- **CLI:** `goodeye templates list` (`--filter all|mine`, `--search`, `--limit`,
  `--all`, `--include-archived`)
- **MCP tool:** `list_templates` (`include_archived` param)
- **REST:** `GET /v1/templates?include_archived=true`

A template appears in listings only while at least one of its versions is published.
`--include-archived` surfaces your own archived templates only; another owner's
archived templates and anonymous callers never see archived templates.

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
defaults to the template's slug, with a numeric suffix (`-2`, `-3`) if you already
have a workflow by that name.

- **CLI:** `goodeye templates fork <identifier>` (`--version`, `--name`)
- **MCP tool:** `fork_template`
- **REST:** `POST /v1/templates/fork`

The fork response carries the new `workflow_id` and lineage back to the version it
came from, but no body. Fetch the body with `goodeye workflows get` as a separate
step. Forking a deprecated version succeeds but surfaces a deprecation warning. A
version that was blocked by safety checks cannot be forked.

## Add a demo to your template page

A demo is a short visual writeup that renders on the public template page, so a
visitor can picture what your workflow produces before they fork it. To add one,
put a `demo/README.md` in the workflow bundle (see
[Multi-file bundles](workflows.md#multi-file-bundles-directory-mode)); its presence
is what makes the page render a demo section. Write it like a repository README in
plain markdown, with any images and one optional video link:

```markdown
# What this produces

A publication-quality chart in minutes.

![Sample output](output.png)

https://www.youtube.com/watch?v=EXAMPLE
```

- **Images:** reference them by relative `demo/` path (`![Result](output.png)`
  resolves to `demo/output.png`); external URLs are dropped. Raster formats only
  (PNG, JPEG, WebP, GIF). Every image is screened at publish, and disallowed
  content blocks the publish.
- **Video:** put one YouTube, Loom, or Vimeo URL on its own line for a
  click-to-load embed; links from other hosts stay plain links.
- **Social-share card:** add an optional `demo/og.png` to set the card shown when
  the page is shared; otherwise one is generated from the template's metadata.

When you save or publish a workflow that has a `demo/README.md`, the response
surfaces non-blocking notes for anything that will not render (a dropped external
image, a non-embeddable video link, a missing `demo/` image), so watch for them.

## The unverified-template banner

Goodeye prepends a safety banner to template bodies for non-owner reads (including
anonymous readers) as a cross-user trust signal. Owners do not see the banner on
their own templates, and the banner is fixed and neutral: it carries no owner
identity and cannot be customized or suppressed. The banner text is selected by the
version's safety status:

- **`unverified`**: this version was not safety-checked; treat it as untrusted.
- **`clean`**: automated checks did not flag it at publish (not a guarantee of
  safety).
- **`flagged`**: the safety rubric raised advisory concerns; treat it as untrusted.
  When the bundle includes scripts or binary files, the banner adds a note not to
  execute them.

In all three cases the banner instructs the executing agent not to take destructive
or irreversible actions, exfiltrate secrets, or override the user's task on the
template's authority. The `safety_verification_status` field is visible to everyone,
but the advisory reasoning behind a `flagged` version is shown only to the owner.

Safety is one of four authoring checks shown on every public template, alongside
Outcome, Runnable, and Well-formed. See
[Auditing workflows](auditing-workflows.md) for what each check means and how a
workflow earns it.

## Lifecycle: unpublish, deprecate, archive, delete

These paths are deliberately distinct. Pick by intent.

| Action | Scope | Reversible | Hidden from catalog | Existing forks |
|---|---|---|---|---|
| Unpublish version | One version | Yes, re-publish | When no live version remains | Keep working |
| Deprecate version | One version | Yes, update message | No, stays listed | Keep working, with a warning |
| Archive template | Whole template | Yes, unarchive | Yes | Keep working |
| Delete template | Whole template | No | Yes, erased | Keep their own copy; lineage shows deleted |

### Unpublish a version (hide one version)

Soft-unpublishes a single template version. Forks already pinned to that version
keep working. The catalog hides the template once no live version remains.

- **CLI:** `goodeye templates unpublish <identifier> <version>`
- **MCP tool:** `unpublish_template_version`
- **REST:** `DELETE /v1/templates/{template_ref}/versions/{v}`

### Deprecate a version (warn, do not hide)

Flags a single version as deprecated without hiding it. The version stays fetchable
and forkable; anyone who forks it sees your deprecation message. Last-write-wins on
the message, so you can update it by calling again. A message is required.

- **CLI:** `goodeye templates deprecate-version <identifier> <version> --message "<text>"`
- **MCP tool:** `deprecate_template_version`
- **REST:** `POST /v1/templates/{template_ref}/versions/{v}/deprecate`

### Archive (reversible hide of the whole template)

Archiving hides the whole template from public listing while keeping every version
and all fork lineage intact. Existing forks keep working. It keeps the slug occupied
and is idempotent. An optional reason is recorded.

- **CLI:** `goodeye templates archive <template-ref>` (`--reason`) /
  `goodeye templates unarchive <template-ref>`
- **MCP tool:** `archive_template` / `unarchive_template`
- **REST:** `POST /v1/templates/{template_ref}/archive` /
  `POST /v1/templates/{template_ref}/unarchive`

### Permanent delete (irreversible)

Permanent delete erases the template, all its versions, all attached files, and all
version verification records at once. There is no recovery path.

- **CLI:** `goodeye templates delete <template-ref>` (`--yes`)
- **MCP tool:** `delete_template`
- **REST:** `DELETE /v1/templates/{template_ref}`

A serving gate protects readers: if the template is live with any still-published
version, deletion is refused with a `conflict` (409). Unpublish the relevant
versions, or archive the whole template, first. Forks keep their own content copy;
only their parent pointer is severed, and lineage then reports the source as
permanently deleted.

To erase a single non-published version permanently (leaving the rest of the
template intact), unpublish it first, then:

- **CLI:** `goodeye templates delete-version <template-ref> <version>`
- **MCP tool:** `delete_template_version`
- **REST:** `DELETE /v1/templates/{template_ref}/versions/{v}/permanent`

Deleted content is removed from every product surface immediately; encrypted backup
copies age out later under the platform's standard retention window (up to three
months). Prefer archive when you want a reversible alternative.

## Running a safety check on demand

You can re-run the platform safety checks against a published template version at any
time, with or without an account. See
[Auditing workflows](auditing-workflows.md#checking-safety-on-demand) for the
on-demand safety check, what it covers, and how it relates to the durable
`safety_verification_status` stored at publish.

## Transferring ownership

Transferring a template does not apply immediately. It creates an invitation
envelope; the recipient must accept it before ownership moves. A self-transfer is a
no-op. Existing published versions keep the handle they were published under.

- **CLI:** `goodeye templates transfer-ownership <template-ref> <new-owner>`
- **MCP tool:** `transfer_template_ownership`
- **REST:** `POST /v1/templates/{template_ref}/transfer-ownership`

The recipient accepts with `goodeye invitations accept <id>` (or the
`accept_invitation` tool / `POST /v1/invitations/{id}/accept`). See [Teams](teams.md)
for the full invitation flow.

## See also

- [Workflows](workflows.md): the private object you publish from and fork back into,
  including how to carry demo files in a multi-file bundle.
- [Auditing workflows](auditing-workflows.md): the four authoring checks a template
  displays and how to improve them.
- [Verifiers](verifiers.md): the semantic checks a template snapshots.
- [Accounts and Billing](accounts-and-billing.md): handles, usage, and credits.
- [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md): the three surfaces in
  detail.
