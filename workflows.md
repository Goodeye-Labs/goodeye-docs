# Workflows

A workflow is the private stored object in Goodeye: a markdown runbook with a
`name`, a one-line `description`, a declared `outcome`, and optional `tags`. It is
where you build and improve the work an agent does for an outcome, in private,
before you ever share it. Workflows stay private to their owner; public sharing
happens on a separate surface, where you publish a snapshot as a
[template](templates.md). There is no visibility switch on a workflow itself.

When an agent fetches a workflow body, it executes that body as your runbook
rather than summarizing it (see
[the agent contract](overview.md#the-agent-contract)).

This page follows the private lifecycle: design and save, version safely, attach
verifiers and image generators, fetch and sync, improve against the outcome, and
share through grants.

## Designing a workflow

A guided design session produces a workflow plus its verifiers. Like every guided
session in Goodeye (design here, and teach and optimize below), it returns a
prompt pack rather than doing the work for you: your agent runs the session
locally with you and calls `save_workflow` when you approve the result. Nothing is
persisted until that save, and design requires an authenticated caller.

- **CLI:** `goodeye design` (pipe it into your agent: `goodeye design > prompt.md`)
- **MCP tool:** `design_workflow`
- **REST:** `GET /v1/design/workflow-prompt`

## Saving and versioning a workflow

Saving creates a workflow on first call and appends a new version on each
subsequent call. A workflow is always private to the caller.

- **CLI:** `goodeye workflows publish <FILE>`
- **MCP tool:** `save_workflow`
- **REST:** `POST /v1/workflows`

The `publish` command takes a markdown file, a directory (see
[Multi-file bundles](#multi-file-bundles-directory-mode) below), or `-` to read
markdown from stdin (preferred for generated agent output). `name`, `description`,
and `outcome` are required, supplied either as YAML front-matter in the body or as
flags; flags win when both are present. Tags are optional.

```sh
goodeye workflows publish - \
  --name high-signal-chart \
  --description "Produce a publication-quality chart on a topic, gated by a design verifier." \
  --outcome "Engagement on the charts we publish." \
  --tag data --tag viz
```

Front-matter form, equivalent to the flags above:

```
---
name: high-signal-chart
description: Produce a publication-quality chart on a topic, gated by a design verifier.
outcome: Engagement on the charts we publish.
tags: [data, viz]
---
```

### Updating safely

The first save creates version 1 and returns a `version_token`. To update an
existing workflow, pass the current token with `--expected-version-token` (the MCP
and REST surfaces take `expected_version_token`). If the token does not match the
current one, the save is rejected with a `conflict` (409) so a stale writer never
clobbers a newer version. Omit the token only when creating a brand-new workflow.

**Note:** if the slug you are saving is held by one of your own archived
workflows, the save is rejected with a `conflict` (409) telling you to unarchive
and rename, or delete, that archived workflow first.

### Verifier references

Bind a deployed semantic verifier to the workflow with a repeatable
`--verifier name=verifier_id` flag (the MCP and REST surfaces accept a structured
`verifiers` array). Each binding name must be lowercase letters, digits, and
hyphens. Append a version (`name=verifier_id@version`) to pin it, which keeps a
published template's verifier snapshot deterministic; an unpinned binding resolves
to the verifier's current version at publish time. On an update, omitting verifiers
preserves the prior set, while `--clear-verifiers` (an empty list) removes them
all. Only the workflow owner can change verifier references. See
[Verifiers](verifiers.md) for deploying them.

### Image generator references

Bind a deployed image generator with a repeatable
`--image-generator name=generator_ref` flag (the MCP and REST surfaces accept a
structured `image_generators` array). The `generator_ref` is a quality tier
(`system:<tier>`), a deployed generator UUID, or `uuid@version` to pin a version.
Binding names follow the same rule as verifiers, and the same update semantics
apply: omitting image generators preserves the prior set, while
`--clear-image-generators` removes them all. Only the workflow owner can change
image generator references. See [Image generators](image-generators.md) for
deploying them.

### Multi-file bundles (directory mode)

A workflow can carry sibling files (scripts, reference docs, assets) alongside its
`SKILL.md` body. Pass a directory that contains a `SKILL.md`: `SKILL.md` becomes
the workflow body and every other non-ignored file is uploaded as a sibling.

```sh
goodeye workflows publish ./high-signal-chart
```

The MCP and REST surfaces carry the tree in a `files` array on the same
`save_workflow` / `POST /v1/workflows` call. The snapshot rule on an update:
omitting `files` carries the prior version's tree forward unchanged (so a
body-only or stdin save never drops siblings); sending `files: []` (or
`--clear-files`) clears the tree; sending a non-empty list snapshots exactly those
files. Common build artifacts (`.git/`, `node_modules/`, `__pycache__/`, `.venv/`,
`dist/`, `build/`, and similar) are ignored by default, and per-file and total
size caps apply.

## Importing an existing skill

Agent skills on disk (under `~/.claude/skills/`, `~/.agents/skills/`, or
`~/.cursor/skills/`) are already directory-shaped: a `SKILL.md` plus optional
sibling files. That is exactly what directory-mode `publish` expects, so importing
one is a single `publish` call (there is no separate import command). Goodeye also
requires an `outcome`, which a skill file lacks, so supply it:

```sh
goodeye workflows publish ~/.claude/skills/high-signal-chart \
  --outcome "Engagement on the charts we publish."
```

`SKILL.md` becomes the workflow body and the other files upload alongside it.
Front-matter keys Goodeye does not recognize (such as `allowed-tools`) are
preserved verbatim in the stored body. To bring over a whole library, run one
`publish` per skill, each with its own real `outcome` (the outcome is what makes a
workflow discoverable, so do not reuse a placeholder). An imported skill lands as
a private workflow; to keep workflows mirrored back to a skills directory on an
ongoing basis, use [sync](#syncing-a-bundle-locally).

## Fetching, listing, and searching

### Get a workflow

Fetch a workflow by UUID or slug. The default response is the markdown body
(wrapped with agent-facing markers so the calling agent knows to execute it); pass
`--json` for the full record, which includes the file manifest, the top-level
`safety_verification_status`, and `archived_at`.

- **CLI:** `goodeye workflows get <id-or-name>` (`--version`, `--output`, `--json`)
- **MCP tool:** `get_workflow` (plus `get_workflow_file`, `get_workflow_files` for
  individual sibling files)
- **REST:** `GET /v1/workflows/{id_or_slug}` (files via
  `GET /v1/workflows/{id_or_slug}/files?path=` or `?paths=`)

The caller must own the workflow or hold a grant on it; otherwise the read masks
as `not_found` (404). Grantee reads are version-floor scoped: a grantee cannot
read a version below their grant's floor. Owners can fetch their own archived
workflows; non-owners receive `not_found` for an archived workflow.

### List workflows

- **CLI:** `goodeye workflows list` (`--filter mine|shared-with-me|all`, `--tag`,
  `--search`, `--limit`, `--all`, `--include-archived`)
- **MCP tool:** `list_workflows` (`include_archived` param)
- **REST:** `GET /v1/workflows?include_archived=true`

Archived workflows are hidden by default. `--include-archived` surfaces your own
archived workflows (archived workflows shared with you stay hidden, since only the
owner can restore one).

### Search workflows

Natural-language, LLM-ranked search over the workflows you own or can access,
distinct from the lexical filtering on `list`.

- **CLI:** `goodeye workflows search "<query>"` (`--filter`, `--tag`, `--limit`)
- **MCP tool:** `search_workflows`
- **REST:** `POST /v1/workflows/search`

## Syncing a bundle locally

Sync mirrors your workflows into local directories (`<target>/<slug>/SKILL.md`
plus sibling files) and reconciles drift in both directions. It is CLI-only and
requires authentication.

- **Add a target:** `goodeye workflows sync target add <path>` (or `--preset
  claude|agents|cursor`, with `--scope owned|all|selected`).
- **Pull to disk:** `goodeye workflows sync pull` (`--force` overwrites local
  edits; `--yes` skips the prompt before removing a workflow deleted upstream).
- **Check drift:** `goodeye workflows sync status`.
- **Push local edits back:** `goodeye workflows sync push`. Each push is
  optimistic-locked; a workflow that moved upstream is reported as a conflict and
  left untouched (reconcile with `pull` first), and a `view`-only workflow is never
  uploaded.

Running `goodeye workflows sync` with no subcommand pulls every target, then
prints status.

**Automatic pull (opt-in):** keep targets fresh without a manual pull. When on,
the CLI pulls new and updated workflows in the background after your command
finishes; it never pushes, never clobbers local edits, and never removes a
directory (those stay manual and explicit). Toggle with
`goodeye workflows sync auto on` (`--interval <seconds>`, default 3600),
`... auto off`, or `... auto` to see the current setting.

## Improving a workflow

Saving a workflow is the start, not the finish. Three guided sessions improve it
against its own outcome. Each works like design above: it returns a prompt pack
your agent runs locally, and persists the winner only after you approve it (the
loop never auto-saves). Each requires at least `edit` access.

**Teach** folds your feedback on real runs back into the workflow.

- **CLI:** `goodeye workflows teach <id-or-name>`
- **MCP tool:** `teach_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/teach`

**Optimize** runs an iteration loop over a locked scenario set, scoring
per-scenario verifier pass rates and proposing a stronger version.
`--max-iterations` defaults to 20.

- **CLI:** `goodeye workflows optimize <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/optimize`

**Optimize the trigger description** tunes the `description` that decides when a
workflow fires, so it triggers on the prompts it should and not on look-alikes. It
changes only the `description` (body, outcome, tags, and files carry forward) and
runs without drawing on your credits. `--max-iterations` defaults to 10.

- **CLI:** `goodeye workflows optimize-description <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_description`
- **REST:** `POST /v1/workflows/{id_or_slug}/optimize-description`

## Auditing and checking safety

To grade a workflow against the authoring checks and apply targeted fixes, or to
run a safety check on demand, see [Auditing workflows](auditing-workflows.md). The
audit reports the same checks every public template displays, so you can improve a
workflow before or after you publish it.

## Archive, unarchive, and permanent delete

These are two distinct paths: a reversible hide (archive) and an irreversible
erase (delete).

### Archive (reversible)

Archiving hides a workflow from list results and grants but keeps every version
and file intact. It keeps the slug occupied, so no new workflow can reuse the name
until you unarchive or delete. Idempotent.

- **CLI:** `goodeye workflows archive <id-or-name>` /
  `goodeye workflows unarchive <id-or-name>`
- **MCP tool:** `archive_workflow` / `unarchive_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/archive` /
  `POST /v1/workflows/{id_or_slug}/unarchive`

Unarchive clears `archived_at` and re-derives the verifier grants that archiving
removed, so it is a faithful inverse.

### Permanent delete (irreversible)

Permanent delete erases the workflow, all its versions, all attached files, and
all access grants at once. There is no recovery path. It works on both live and
archived workflows.

- **CLI:** `goodeye workflows delete <id-or-name>` (`--yes` to skip the prompt)
- **MCP tool:** `delete_workflow`
- **REST:** `DELETE /v1/workflows/{id_or_slug}`

Deleted content is removed from every product surface immediately; backup copies
age out later under the standard retention window (see
[Accounts and Billing](accounts-and-billing.md)). Prefer archive when you want a
reversible alternative.

### Delete a single version

Erase one non-current version permanently. The current (live) version cannot be
erased this way; use `delete_workflow` to remove everything. Version numbers stay
monotonic with a gap where the erased version was, and surviving versions are not
renumbered.

- **CLI:** `goodeye workflows delete-version <id-or-name> <version>`
- **MCP tool:** `delete_workflow_version`
- **REST:** `DELETE /v1/workflows/{id_or_slug}/versions/{n}`

## Sharing with grants

A grant gives a named user or team access to a private workflow. There are three
roles, in increasing order of capability:

- **view**: read the workflow, run it, and audit it.
- **edit**: also save new versions and teach or optimize.
- **admin**: also manage grants.

Grant access:

- **CLI:** `goodeye workflows grant <id-or-name> <grantee> <role>` (`--include-history`)
- **MCP tool:** `grant_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/grants`

The grantee is a user or team UUID, an email, or a `@handle`. You cannot grant
above your own role. Grants are rate-limited per day per granter.

### Version-floor scoping

By default a grant is floored at the version current when you shared it: the
grantee sees that version and later ones, never earlier history. Pass
`--include-history` (`include_history=true`) to share the full version history
instead. A later plain role change does not re-scope what a grantee can already
see. When a user holds several grants (direct plus team), the most permissive
floor wins.

### Verifier grants cascade

When you grant a workflow, the semantic verifiers it references are shared with the
same grantee automatically, scoped to that workflow: a `view` grant lets them read
and run those verifiers, and `edit` or `admin` also lets them deploy new verifier
versions. Changing the workflow's verifier references on a later save updates the
cascaded access to match, and revoking the workflow grant removes it.

### List and revoke grants

- **CLI:** `goodeye workflows grants <id-or-name>` /
  `goodeye workflows revoke-grant <id-or-name> <grantee>`
- **MCP tool:** `list_workflow_grants` / `revoke_workflow_grant`
- **REST:** `GET /v1/workflows/{id_or_slug}/grants` /
  `DELETE /v1/workflows/{id_or_slug}/grants`

The grants listing shows each grantee by `@handle` (non-self emails are redacted),
the role, whether it arrived via a team, and the history scope.

### Leaving a shared workflow

If a workflow was shared with you, you can drop your own direct grant. Owners
cannot leave their own workflow.

- **CLI:** `goodeye workflows leave <id-or-name>`
- **MCP tool:** `leave_shared_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/leave`

## Transferring ownership

Transferring a workflow does not apply immediately. It creates an invitation
envelope; the recipient must accept it before ownership and the verifier
dependencies move over. A self-transfer is a no-op.

- **CLI:** `goodeye workflows transfer-ownership <id-or-name> <new-owner>`
- **MCP tool:** `transfer_workflow_ownership`
- **REST:** `POST /v1/workflows/{id_or_slug}/transfer-ownership`

The recipient accepts with `goodeye invitations accept <id>` (or the
`accept_invitation` tool / `POST /v1/invitations/{id}/accept`). See [Teams](teams.md)
for the full invitation flow.

## Fork lineage

A workflow created by forking a [template](templates.md) carries lineage back to
the exact template version it came from. Lineage reports the parent template, the
pinned version, the upstream latest version, and whether the upstream was later
archived, had the pinned version deprecated, or was permanently deleted. A fork
keeps its own content copy: if the parent template is permanently deleted, the fork
still works and lineage reports the source as permanently deleted.

- **CLI:** `goodeye workflows lineage <id-or-name>` (also
  `goodeye templates lineage <id-or-name>`, same view)
- **MCP tool:** `lookup_fork_lineage`
- **REST:** `GET /v1/workflows/{id_or_slug}/lineage`

## See also

- [Verifiers](verifiers.md): deploy the semantic checks a workflow references.
- [Auditing workflows](auditing-workflows.md): grade a workflow against the
  authoring checks.
- [Templates](templates.md): publish a workflow publicly, fork one back, and manage
  the public lifecycle.
- [Teams](teams.md): grants, teams, and the invitation flow.
- [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md): the three surfaces in
  detail.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, and usage.
