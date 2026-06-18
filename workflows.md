# Workflows

A workflow is the private stored object in Goodeye: a markdown runbook with a
`name`, a one-line `description`, a declared `outcome`, and optional `tags`. This
page covers the full private lifecycle: designing, saving, versioning, fetching,
syncing, archiving, deleting, teaching, optimizing, auditing, sharing through
grants, transferring, and tracing fork lineage.

Workflows stay private to their owner. Public sharing happens on a separate
surface: you publish a snapshot of a workflow as a [template](templates.md).
There is no visibility switch on a workflow itself.

**The agent contract:** when an agent fetches a workflow body, it executes that
body as your runbook. It follows the instructions itself rather than summarizing
or just displaying them. See [Overview](overview.md) for the full mental model.

## Designing a workflow

Start a guided design session to produce a workflow plus its verifiers. The
design surface returns a prompt pack (a `SKILL.md` plus reference files); your
agent follows it locally with you, then calls `save_workflow` when the design is
finalized. Nothing is persisted by the design call itself, and design requires
an authenticated caller.

- **CLI:** `goodeye design` (pipe it into your agent: `goodeye design > prompt.md`)
- **MCP tool:** `design_workflow`
- **REST:** `GET /v1/design/workflow-prompt`

The agent never writes the generated workflow to your local filesystem as the
final step. It calls `save_workflow` so the workflow lands in the registry.

## Saving and versioning a workflow

Saving creates a workflow on first call and appends a new version on each
subsequent call. A workflow is always private to the caller.

- **CLI:** `goodeye workflows publish <FILE>`
- **MCP tool:** `save_workflow`
- **REST:** `POST /v1/workflows`

The `publish` command takes a markdown file, a directory (see
[Multi-file bundles](#multi-file-bundles-directory-mode) below), or `-` to read
markdown from stdin (preferred for generated agent output). `name`,
`description`, and `outcome` are required, supplied either as YAML front-matter
in the body or as flags; flags win when both are present. Tags are optional.

```sh
goodeye workflows publish - \
  --name incident-postmortem \
  --description "Draft a postmortem from an incident transcript." \
  --outcome "Reduce mean-time-to-postmortem from days to hours." \
  --tag sre --tag postmortem
```

Front-matter form, equivalent to the flags above:

```
---
name: incident-postmortem
description: Draft a postmortem from an incident transcript. Use when ...
outcome: Reduce mean-time-to-postmortem from days to hours.
tags: [sre, postmortem]
---
```

### Optimistic-locking on updates

The first save creates version 1 and returns a `version_token`. To update an
existing workflow, pass the current token with `--expected-version-token` (the
MCP and REST surfaces take `expected_version_token`). If the token does not match
the registry's current token, the save is rejected with a `conflict` (409) so a
stale writer never clobbers a newer version. Omit the token only when creating a
brand-new workflow.

**Note:** if the slug you are saving is held by one of your own archived
workflows, the save is rejected with a `conflict` (409) directing you to
unarchive and rename, or delete, that archived workflow first.

### Verifier references

Bind a deployed semantic verifier to the workflow with a repeatable
`--verifier name=verifier_id` flag (the MCP and REST surfaces accept a structured
`verifiers` array). Each binding name must be lowercase letters, digits, and
hyphens. Append a version to the binding (`name=verifier_id@version`) to pin it,
which keeps a published template's verifier snapshot deterministic; an unpinned
binding resolves to the verifier's current version at publish time. Use
`--clear-verifiers` to send an empty binding list and remove all existing
bindings. On an update, omitting verifiers entirely preserves the prior set;
sending an explicit empty list clears them. Only the workflow owner can change
verifier references. See [Verifiers](verifiers.md) for deploying them.

### Image generator references

Bind a deployed image generator to the workflow with a repeatable
`--image-generator name=generator_ref` flag (the MCP and REST surfaces accept a
structured `image_generators` array). The `generator_ref` is a quality tier
(`system:<tier>`), a deployed generator UUID, or `uuid@version` to pin a specific
version, which keeps a published template's generator snapshot deterministic.
Binding names follow the same rule as verifiers (lowercase letters, digits, and
hyphens), and the same update semantics apply: omitting image generators
preserves the prior set, while `--clear-image-generators` sends an empty list to
remove them all. Only the workflow owner can change image generator references.
See [Image generators](image-generators.md) for deploying them.

### Multi-file bundles (directory mode)

A workflow can carry sibling files (scripts, reference docs, assets) alongside
its `SKILL.md` body. Pass a directory that contains a `SKILL.md`: `SKILL.md`
becomes the workflow body and every other non-ignored file is uploaded as a
sibling.

```sh
goodeye workflows publish ./incident-postmortem
```

The MCP and REST surfaces carry the tree in a `files` array on the same
`save_workflow` / `POST /v1/workflows` call.

The snapshot rule for the file tree on an update:

- **Omitting `files` carries the prior version's tree forward** unchanged (a
  body-only save never drops siblings). A single-file or stdin `publish` omits
  files, so the server preserves the existing tree.
- **Sending `files: []` clears the tree.** From the CLI, `--clear-files` on a
  single-file or stdin input does this; it cannot be combined with a directory
  input.
- **Sending a non-empty list snapshots exactly those files.** Directory mode
  uploads the full tree this way.

Common build artifacts are ignored by default (`.git/`, `node_modules/`,
`__pycache__/`, `.venv/`, `.DS_Store`, `dist/`, `build/`, `*.log`, and similar).
Per-file and total size caps apply.

### Importing an existing skill

If you already keep agent skills on disk (for example under `~/.claude/skills/`,
`~/.agents/skills/`, or `~/.cursor/skills/`), each one is a directory holding a
`SKILL.md` plus optional sibling files. That is exactly the shape directory-mode
publish expects, so importing a skill into Goodeye is a single `publish` call.
There is no separate import command.

A skill's `SKILL.md` front-matter already carries a `name` and a `description`.
Goodeye also requires an `outcome`, the measurable result the workflow drives
toward, which a skill file does not have. Supply it with `--outcome`, or add an
`outcome:` line to the front-matter before importing:

```sh
goodeye workflows publish ~/.claude/skills/incident-postmortem \
  --outcome "Reduce mean-time-to-postmortem from days to hours."
```

`SKILL.md` becomes the workflow body and every other non-ignored file in the
directory uploads alongside it. Front-matter keys Goodeye does not recognize
(such as `allowed-tools`) are preserved verbatim in the stored body; they are
simply not read as discovery facets. The default ignore rules and size caps from
[directory mode](#multi-file-bundles-directory-mode) apply.

To bring over a whole library, run one `publish` per skill, each with its own
outcome. The outcome is what makes a workflow discoverable, so set a real one for
each rather than reusing a single placeholder:

```sh
for skill in ~/.claude/skills/*/; do
  # Replace the outcome per skill before running.
  goodeye workflows publish "$skill" --outcome "State the measurable outcome here"
done
```

The same import works on the other surfaces, matching three-surface parity: an
agent connected over [MCP](mcp.md) reads the skill's files locally and calls
`save_workflow` with a `files` array (plus an `outcome`), and the
[REST API](rest-api.md) takes the same tree on `POST /v1/workflows`.

An imported skill lands as a private workflow, like any other save. To share it
publicly afterward, publish it as a [template](templates.md). To keep registry
workflows mirrored back to a skills directory on an ongoing basis (the reverse
direction), use [sync](#syncing-a-bundle-locally) rather than a one-time import.

## Fetching, listing, and searching

### Get a workflow

Fetch a workflow by UUID or slug. The default response is the markdown body
(wrapped with agent-facing markers so the calling agent knows to execute it);
pass `--json` for the full record, which includes the file manifest, the
top-level `safety_verification_status`, and `archived_at`.

- **CLI:** `goodeye workflows get <id-or-name>` (`--version`, `--output`, `--json`)
- **MCP tool:** `get_workflow` (plus `get_workflow_file`, `get_workflow_files`
  for individual sibling files)
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

The sync commands mirror your registry workflows into local directories
(`<target>/<slug>/SKILL.md` plus sibling files) and reconcile drift in both
directions. They are CLI-only, and they require authentication.

1. **Add a target directory:** `goodeye workflows sync target add <path>`
   (or `--preset claude|agents|cursor`, with `--scope owned|all|selected`).
2. **Pull registry workflows to disk:** `goodeye workflows sync pull` (optional
   slugs to limit scope; `--force` overwrites local edits; `--yes` skips the
   prompt before removing a workflow deleted on the registry).
3. **Check drift without touching anything:** `goodeye workflows sync status`.
4. **Push locally edited workflows back:** `goodeye workflows sync push`. Each
   push sends the full directory tree and is optimistic-locked; if the registry
   moved since the last sync, the workflow is reported as a conflict and left
   untouched (reconcile with `pull` first). Renaming through push is not
   supported, and a workflow you hold only a `view` grant on is never uploaded.

Running `goodeye workflows sync` with no subcommand pulls every target and then
prints status. `goodeye workflows sync` and `goodeye workflows pull` consume the
file-fetch routes internally; you do not call them directly.

### Automatic pull

The automatic pull is an opt-in mode that keeps your configured sync targets
fresh without a manual `pull`. When enabled, the CLI pulls the safe set (new
workflows and ones updated on the registry) in the background after your command
completes, so your local copies stay current for whatever agent loads them.

Turn it on with:

```sh
goodeye workflows sync auto on                    # enable with the default interval (1 hour)
goodeye workflows sync auto on --interval 1800    # enable with a custom interval in seconds
goodeye workflows sync auto off                   # disable
goodeye workflows sync auto                       # print current setting and last run time
```

What the automatic pull does and does not do:

- **Writes** new workflows and ones where the registry version has moved since
  the last sync. Only those workflows are fetched; unchanged ones are skipped.
- **Skips, never clobbers** workflows with local edits or conflicts. These are
  surfaced in a one-line notice so you know to reconcile them manually with
  `workflows sync pull` or `workflows sync status`.
- **Reports, never removes** workflows that were deleted on the registry. Auto
  mode never removes a local directory; deletion is always an explicit, confirmed
  step through a manual `pull`.
- **Never pushes.** The automatic path is pull-only. Pushing local edits back
  is always an explicit `workflows sync push`.

The automatic pull is global: it applies to every configured target. The
interval (default 3600 seconds) is a floor on how often the pull runs, not a
freshness guarantee; a manual `pull` still gives you an immediate refresh.

The pull runs after your command finishes and never delays it. When it writes,
skips, or finds a workflow gone from the registry, it prints a single line to
stderr. When nothing changed and nothing was skipped or gone, it stays silent. It is suppressed in CI environments, for
`--json` output, for `--help`/`--version` and the bare invocation, and when you
are explicitly running a `workflows sync` command, so it never shadows an
operation you ran yourself.

## Archive, unarchive, and permanent delete

These are two distinct paths: a reversible hide (archive) and an irreversible
erase (delete).

### Archive (reversible)

Archiving hides a workflow from list results and grants but keeps every version
and file intact. It keeps the slug occupied, so no new workflow can reuse the
name until you unarchive or delete. Idempotent.

- **CLI:** `goodeye workflows archive <id-or-name>` / `goodeye workflows unarchive <id-or-name>`
- **MCP tool:** `archive_workflow` / `unarchive_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/archive` /
  `POST /v1/workflows/{id_or_slug}/unarchive`

Unarchive clears `archived_at` and re-derives the verifier grants that archiving
removed, so it is a faithful inverse.

### Permanent delete (irreversible)

Permanent delete erases the workflow, all its versions, all attached files, and
all access grants from the live system at once. There is no recovery path. It
works on both live and archived workflows.

- **CLI:** `goodeye workflows delete <id-or-name>` (`--yes` to skip the prompt)
- **MCP tool:** `delete_workflow`
- **REST:** `DELETE /v1/workflows/{id_or_slug}`

**Note:** encrypted backups age out within the platform's standard retention
window (up to three months), so deleted content is not instantly erased from all
systems everywhere, but it is no longer reachable through any product surface.
Prefer archive when you want a reversible alternative.

### Delete a single version

Erase one non-current version permanently. The current (live) version cannot be
erased this way; use `delete_workflow` to remove everything. Version numbers stay
monotonic with a gap where the erased version was, and surviving versions are not
renumbered.

- **CLI:** `goodeye workflows delete-version <id-or-name> <version>`
- **MCP tool:** `delete_workflow_version`
- **REST:** `DELETE /v1/workflows/{id_or_slug}/versions/{n}`

## Checking safety

Run the platform safety checks against a saved workflow version on demand. This
writes no state; it bills two metered verifier runs and returns a `status` of
`clean`, `flagged`, `blocked`, or `error`. Use it to preview what a
[template publish](templates.md#publishing-a-version) would compute.

- **CLI:** `goodeye workflows check-safety <id-or-name>` (`--version`, `--json`)
- **MCP tool:** `check_workflow_safety`
- **REST:** `POST /v1/workflows/{id_or_slug}/safety-check`

## Teaching a workflow

Teaching improves an existing workflow from your feedback. Like design, it
returns a prompt pack; your agent runs the session locally and persists the
result with `save_workflow` (marking the provenance `source=teach`). Requires at
least `edit` access on the workflow.

- **CLI:** `goodeye workflows teach <id-or-name>`
- **MCP tool:** `teach_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/teach`

## Optimizing a workflow

Optimization runs an agent-driven iteration loop over a locked scenario set,
scoring per-scenario per-verifier pass rates, and persists the winner only after
you explicitly approve it (the loop never auto-saves). Like teach, it returns a
prompt pack and requires at least `edit` access. `max_iterations` defaults to 20
and accepts 1 to 1000 (the upper bound is a safety cap, not a recommended
setting).

- **CLI:** `goodeye workflows optimize <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/optimize`

## Optimizing a workflow's trigger description

A workflow's `description` is the text that decides when it fires. A weak
description misfires: it triggers on prompts it should not, or fails to trigger
on prompts it should. Description optimization tunes that text for trigger
accuracy. Like the other packs, it returns a prompt pack your agent runs
locally; it changes only the `description` (the body, outcome, tags, and any
attached files carry forward unchanged), and it persists the winner only after
you explicitly approve it (the loop never auto-saves). Requires at least `edit`
access. `max_iterations` defaults to 10 and accepts 1 to 1000 (the upper bound
is a safety cap, not a recommended setting).

The loop builds a labeled set of test prompts (the ones the workflow should fire
on, and look-alikes it should not), then iterates: it proposes a sharper
description, re-checks every test prompt with an isolated firing test, and keeps
the change only when it improves how reliably the workflow triggers. It does not
draw from your credit balance, because the firing test is a routing judgment your
own agent makes rather than a metered verifier run. The saved version is marked
with the provenance `source=description_optimization`.

- **CLI:** `goodeye workflows optimize-description <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_description`
- **REST:** `POST /v1/workflows/{id_or_slug}/optimize-description`

## Auditing a workflow

An audit is an opinionated health check against a documented best-practice
rubric. Like the other packs, it returns a prompt pack your agent runs locally.
It inspects the workflow body and every directing sibling file, enforces at least
one criterion with a platform LLM judge, and produces a priority-ranked report:
each finding is tagged P0 (blocker), P1 (recommended), or P2 (polish), quotes the
text to change, and states one concrete fix. Requires at least `view` access.

The rubric covers both workflow design (a specific, measurable outcome,
outcome-specific verifiers with explicit pass and fail conditions, a clear input
and output contract) and skill authoring (actionable instructions, consistent
terminology, a discoverable description). You choose which findings to fix; the
audit applies only what you approve and never auto-saves, editing a local copy
first when one exists and marking the saved version `source=audit`. You can also
audit a local skill that is not on Goodeye yet, in which case the report
recommends saving it.

- **CLI:** `goodeye workflows audit <id-or-name>` (omit the id to audit a local skill)
- **MCP tool:** `audit_workflow`
- **REST:** `POST /v1/workflows/{id_or_slug}/audit`, or `GET /v1/audit/workflow-prompt` for a local skill

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

When you grant a workflow, the semantic verifiers it references are granted to
the same grantee automatically, scoped to that workflow: a `view` grant cascades
run-only verifier access, while `edit` and `admin` cascade tune access. Changing
the workflow's verifier references on a later save updates the cascaded grants to
match, and revoking the workflow grant removes the cascaded verifier grants.

### List and revoke grants

- **CLI:** `goodeye workflows grants <id-or-name>` /
  `goodeye workflows revoke-grant <id-or-name> <grantee>`
- **MCP tool:** `list_workflow_grants` / `revoke_workflow_grant`
- **REST:** `GET /v1/workflows/{id_or_slug}/grants` /
  `DELETE /v1/workflows/{id_or_slug}/grants`

The grants listing shows each grantee by `@handle` (non-self emails are
redacted), the role, whether it arrived via a team, and the history scope (full,
or from a specific version).

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
`accept_invitation` tool / `POST /v1/invitations/{id}/accept`). See
[Teams](teams.md) for the full invitation flow.

## Fork lineage

A workflow created by forking a [template](templates.md) carries lineage back to
the exact template version it came from. Lineage reports the parent template, the
pinned version, the upstream latest version, and whether the upstream was later
archived, had the pinned version deprecated, or was permanently deleted. A fork
keeps its own content copy: if the parent template is permanently deleted, the
fork still works and lineage reports the source as permanently deleted.

- **CLI:** `goodeye workflows lineage <id-or-name>` (also
  `goodeye templates lineage <id-or-name>`, same view)
- **MCP tool:** `lookup_fork_lineage`
- **REST:** `GET /v1/workflows/{id_or_slug}/lineage`

## See also

- [Templates](templates.md): publish a workflow publicly, fork one back, and
  manage public lifecycle.
- [Verifiers](verifiers.md): deploy the semantic checks a workflow references.
- [Teams](teams.md): grants, teams, and the invitation flow.
- [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md): the three surfaces in
  detail.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, and usage.
