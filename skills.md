# Skills

A skill is the private stored object in Goodeye: a markdown runbook with a
`name`, a one-line `description`, and optional `tags` and `outcome`. It is where
you build and improve the work an agent does for you, in private, before you ever
share it. Skills stay private to their owner. You share one with named people or
teams through a grant, and the verifiers it references travel with it. Public
sharing happens on a separate surface, where you publish a snapshot as a
[template](templates.md). There is no visibility switch on a skill itself.

The hosted skill is the source of truth. [Sync](#syncing-a-bundle-locally)
mirrors it into the directories your tools read, so every machine and every agent
runs the current version, and so does everyone you granted it to.

When an agent fetches a skill body, it executes that body as your runbook rather
than summarizing it (see [the agent contract](overview.md#the-agent-contract)).

This page follows the private lifecycle: design and save, version safely, attach
verifiers and image generators, fetch and sync, improve against real runs, and
share through grants.

## Designing a skill

A guided design session produces a skill plus its verifiers. Like every guided
session in Goodeye (design here, and teach and optimize below), it returns a
prompt pack rather than doing the work for you: your agent runs the session
locally with you and calls `save_skill` when you approve the result. Nothing is
persisted until that save, and design requires an authenticated caller.

- **CLI:** `goodeye design` (pipe it into your agent: `goodeye design > prompt.md`)
- **MCP tool:** `design_skill`
- **REST:** `GET /v1/design/skill-prompt`

## Saving and versioning a skill

Saving creates a skill on first call and appends a new version on each subsequent
call. A skill is always private to the caller.

- **CLI:** `goodeye skills publish <FILE>`
- **MCP tool:** `save_skill`
- **REST:** `POST /v1/skills`

The `publish` command takes a markdown file, a directory (see
[Multi-file bundles](#multi-file-bundles-directory-mode) below), or `-` to read
markdown from stdin (preferred for generated agent output). `name` and
`description` are required, supplied either as YAML front-matter in the body or
as flags; flags win when both are present. Tags and outcome are optional.

```sh
goodeye skills publish - \
  --name high-signal-chart \
  --description "Produce a publication-quality chart on a topic, gated by a design verifier." \
  --tag data --tag viz
```

Front-matter form, equivalent to the flags above:

```
---
name: high-signal-chart
description: Produce a publication-quality chart on a topic, gated by a design verifier.
tags: [data, viz]
---
```

### Optional metadata

Two fields are optional, and neither changes whether a skill saves, runs, or
publishes:

- `tags`: discovery facets you can filter on with `list`.
- `outcome`: a one-line note on the result the skill serves. Set it with
  `--outcome` or a front-matter key when it helps you and your teammates track
  why a skill exists. It is displayed when present and omitted when not.

### Updating safely

The first save creates version 1 and returns a `version_token`. To update an
existing skill, pass the current token with `--expected-version-token` (the MCP
and REST surfaces take `expected_version_token`). If the token does not match the
current one, the save is rejected with a `conflict` (409) so a stale writer never
clobbers a newer version. Omit the token only when creating a brand-new skill.

**Note:** if the slug you are saving is held by one of your own archived skills,
the save is rejected with a `conflict` (409) telling you to unarchive and rename,
or delete, that archived skill first.

### Verifier references

Bind a deployed semantic verifier to the skill with a repeatable
`--verifier name=verifier_id` flag (the MCP and REST surfaces accept a structured
`verifiers` array). Each binding name must be lowercase letters, digits, and
hyphens. Append a version (`name=verifier_id@version`) to pin it, which keeps a
published template's verifier snapshot deterministic; an unpinned binding resolves
to the verifier's current version at publish time. On an update, omitting verifiers
preserves the prior set, while `--clear-verifiers` (an empty list) removes them
all. Only the skill owner can change verifier references; someone editing a skill
shared with them can send the existing references back unchanged, which leaves
them as they are. See [Verifiers](verifiers.md) for deploying them.

### Image generator references

Bind a deployed image generator with a repeatable
`--image-generator name=generator_ref` flag (the MCP and REST surfaces accept a
structured `image_generators` array). The `generator_ref` is a quality tier
(`system:<tier>`), a deployed generator UUID, or `uuid@version` to pin a version.
Binding names follow the same rule as verifiers, and the same update semantics
apply: omitting image generators preserves the prior set, while
`--clear-image-generators` removes them all. Only the skill owner can change
image generator references, with the same allowance for sending the existing
ones back unchanged. See [Image generators](image-generators.md) for
deploying them.

### Multi-file bundles (directory mode)

A skill can carry sibling files (scripts, reference docs, assets) alongside its
`SKILL.md` body. Pass a directory that contains a `SKILL.md`: `SKILL.md` becomes
the skill body and every other non-ignored file is uploaded as a sibling.

```sh
goodeye skills publish ./high-signal-chart
```

The MCP and REST surfaces carry the tree in a `files` array on the same
`save_skill` / `POST /v1/skills` call. The snapshot rule on an update: omitting
`files` carries the prior version's tree forward unchanged (so a body-only or
stdin save never drops siblings); sending `files: []` (or `--clear-files`) clears
the tree; sending a non-empty list snapshots exactly those files. Common build
artifacts (`.git/`, `node_modules/`, `__pycache__/`, `.venv/`, `dist/`, `build/`,
and similar) are ignored by default, and per-file and total size caps apply.

## Importing a skill file from disk

This is the usual way in: most people arrive with a working skill file and want
it hosted rather than starting over.

Agent skill files on disk (under `~/.claude/skills/`, `~/.agents/skills/`, or
`~/.cursor/skills/`) are already directory-shaped: a `SKILL.md` plus optional
sibling files. That is exactly what directory-mode `publish` expects, so importing
one is a single `publish` call. There is no separate import command.

```sh
goodeye skills publish ~/.claude/skills/high-signal-chart
```

`SKILL.md` becomes the hosted skill's body and the other files upload alongside
it. Front-matter keys Goodeye does not recognize (such as `allowed-tools`) are
preserved verbatim in the stored body. To bring over a whole library, run one
`publish` per skill file on disk. An imported skill file lands as a private hosted
skill; to keep your hosted skills mirrored back to a skills directory on disk on
an ongoing basis, use [sync](#syncing-a-bundle-locally).

## Fetching, listing, and searching

### Get a skill

Fetch a skill by UUID or slug. The default response is the markdown body, which
opens with a standing directive telling the calling agent to run the skill rather
than display it. Pass `--output PATH` to write the directive-free body to a file
(the form you can edit and re-publish), or `--json` for the full record, which
includes the file manifest, the top-level `safety_verification_status`, and
`archived_at`.

- **CLI:** `goodeye skills get <id-or-name>` (`--version`, `--output`, `--json`)
- **MCP tool:** `get_skill` (plus `get_skill_file`, `get_skill_files` for
  individual sibling files)
- **REST:** `GET /v1/skills/{id_or_slug}` (files via
  `GET /v1/skills/{id_or_slug}/files?path=` or `?paths=`)

The caller must own the skill or hold a grant on it; otherwise the read masks as
`not_found` (404). Grantee reads are version-floor scoped: a grantee cannot read a
version below their grant's floor. Owners can fetch their own archived skills;
non-owners receive `not_found` for an archived skill.

### List skills

- **CLI:** `goodeye skills list` (`--filter mine|shared-with-me|all`, `--tag`,
  `--search`, `--limit`, `--all`, `--include-archived`)
- **MCP tool:** `list_skills` (`include_archived` param)
- **REST:** `GET /v1/skills?include_archived=true`

Archived skills are hidden by default. `--include-archived` surfaces your own
archived skills (archived skills shared with you stay hidden, since only the owner
can restore one).

### Search skills

Natural-language, LLM-ranked search over the skills you own or can access,
distinct from the lexical filtering on `list`.

- **CLI:** `goodeye skills search "<query>"` (`--filter`, `--tag`, `--limit`)
- **MCP tool:** `search_skills`
- **REST:** `POST /v1/skills/search`

## Syncing a bundle locally

Sync is what keeps one hosted skill current everywhere you work. Agents that load
skill files from disk read whatever is in their directory, so without sync each
machine and each tool drifts into its own copy. Point a target at the directory a
tool reads and the hosted skill stays the source of truth: edit it once, and
Claude Code, Codex, Cursor, and anything else reading those directories pick up
the change on the next pull.

Sync mirrors your hosted skills into local directories as
`<target>/<slug>/SKILL.md` plus sibling files, and reconciles drift between the
skill files on disk and the hosted copies in both directions. It is CLI-only and
requires authentication. The subcommands are `sync target add`, `sync pull`,
`sync status`, `sync push`, and `sync auto`.

Configure a target with a preset for a known location, or a path for anything
else:

```sh
goodeye skills sync target add --preset claude    # ~/.claude/skills
goodeye skills sync target add --preset agents    # ~/.agents/skills
goodeye skills sync target add --preset cursor    # ~/.cursor/skills
goodeye skills sync target add ~/work/skills
```

`--preset codex` is another name for `--preset agents`: Codex reads personal
skills from `~/.agents/skills`, so either name configures the same directory.
Use whichever reads more clearly to you, but configure only one of them, since a
second target on the same directory is rejected as a conflict.

- `sync pull` writes your hosted skills down to skill directories on disk.
- `sync push` uploads a skill file you edited on disk back to its hosted skill.
- `sync status` reports drift between the two without writing anything.

### Automatic pull

The automatic background pull is opt-in: turn it on with `sync auto on` (default
interval one hour). It pulls only new and updated hosted skills after a command
completes. It never overwrites your local edits, deletes skill files on disk, or
pushes, and a local conflict is reported rather than clobbered. It is suppressed
in CI, for machine-readable output, and during a manual sync.

```sh
goodeye skills sync auto on --interval 3600
goodeye skills sync auto off
```

See [CLI](cli.md) for the full sync reference.

## Improving a skill

Saving a skill is the start, not the finish. Three guided sessions improve it
against real runs and its own verifiers. Each works like design above: it returns
a prompt pack your agent runs locally, and persists the winner only after you
approve it (the loop never auto-saves). Each requires at least `edit` access.

### Teaching a skill

**Teach** folds your feedback on real runs back into the skill.

- **CLI:** `goodeye skills teach <id-or-name>`
- **MCP tool:** `teach_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/teach`

### Optimizing a skill

**Optimize** runs an iteration loop over a locked scenario set, scoring
per-scenario verifier pass rates and proposing a stronger version.
`--max-iterations` defaults to 20.

- **CLI:** `goodeye skills optimize <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/optimize`

### Optimizing a trigger description

**Optimize the trigger description** tunes the `description` that decides when a
skill fires, so it triggers on the prompts it should and not on look-alikes. It
changes only the `description` (body, tags, files, and any outcome carry forward)
and runs without drawing on your credits. `--max-iterations` defaults to 10.

- **CLI:** `goodeye skills optimize-description <id-or-name>` (`--max-iterations`)
- **MCP tool:** `optimize_description`
- **REST:** `POST /v1/skills/{id_or_slug}/optimize-description`

## Auditing and checking safety

To grade a skill against the authoring checks and apply targeted fixes, or to run
a safety check on demand, see [Auditing skills](auditing-skills.md). The audit
reports the same checks every public template displays, so you can improve a skill
before or after you publish it.

## Archive, unarchive, and permanent delete

These are two distinct paths: a reversible hide (archive) and an irreversible
erase (delete).

### Archive (reversible)

Archiving hides a skill from list results and grants but keeps every version and
file intact. It keeps the slug occupied, so no new skill can reuse the name until
you unarchive or delete. Idempotent.

- **CLI:** `goodeye skills archive <id-or-name>` /
  `goodeye skills unarchive <id-or-name>`
- **MCP tool:** `archive_skill` / `unarchive_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/archive` /
  `POST /v1/skills/{id_or_slug}/unarchive`

Unarchive clears `archived_at` and re-derives the verifier grants that archiving
removed, so it is a faithful inverse.

### Permanent delete (irreversible)

Permanent delete erases the skill, all its versions, all attached files, and all
access grants at once. There is no recovery path. It works on both live and
archived skills.

- **CLI:** `goodeye skills delete <id-or-name>` (`--yes` to skip the prompt)
- **MCP tool:** `delete_skill`
- **REST:** `DELETE /v1/skills/{id_or_slug}`

Deleted content is removed from every product surface immediately; encrypted
backup copies age out later under the platform's standard retention window (up to
three months). Prefer archive when you want a reversible alternative.

### Delete a single version

Erase one non-current version permanently. The current (live) version cannot be
erased this way; use `delete_skill` to remove everything. Version numbers stay
monotonic with a gap where the erased version was, and surviving versions are not
renumbered.

- **CLI:** `goodeye skills delete-version <id-or-name> <version>`
- **MCP tool:** `delete_skill_version`
- **REST:** `DELETE /v1/skills/{id_or_slug}/versions/{n}`

## Sharing with grants

A grant gives a named user or team access to a private skill. There are three
roles, in increasing order of capability:

- **view**: read the skill, run it, and audit it.
- **edit**: also save new versions and teach or optimize.
- **admin**: also manage grants.

| Capability | view | edit | admin |
|---|---|---|---|
| Read, run, and audit the skill | Yes | Yes | Yes |
| Read and run its semantic verifiers | Yes | Yes | Yes |
| Save new versions; teach and optimize | - | Yes | Yes |
| Deploy new verifier versions | - | Yes | Yes |
| Manage grants on the skill | - | - | Yes |

Grant access:

- **CLI:** `goodeye skills grant <id-or-name> <grantee> <role>` (`--include-history`)
- **MCP tool:** `grant_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/grants`

The grantee is a user or team UUID, an email, or a `@handle`. You cannot grant
above your own role. Grants are rate-limited per day per granter. The grantee is
emailed with a link to the skill.

### Version-floor scoping

By default a grant is floored at the version current when you shared it: the
grantee sees that version and later ones, never earlier history. Pass
`--include-history` (`include_history=true`) to share the full version history
instead. A later plain role change does not re-scope what a grantee can already
see. When a user holds several grants (direct plus team), the most permissive
floor wins.

### Verifier grants cascade

When you grant a skill, the semantic verifiers it references are shared with the
same grantee automatically, scoped to that skill: a `view` grant lets them read
and run those verifiers, and `edit` or `admin` also lets them deploy new verifier
versions. Changing the skill's verifier references on a later save updates the
cascaded access to match, and revoking the skill grant removes it.

### List and revoke grants

- **CLI:** `goodeye skills grants <id-or-name>` /
  `goodeye skills revoke-grant <id-or-name> <grantee>`
- **MCP tool:** `list_skill_grants` / `revoke_skill_grant`
- **REST:** `GET /v1/skills/{id_or_slug}/grants` /
  `DELETE /v1/skills/{id_or_slug}/grants`

The grants listing shows each grantee by `@handle` (non-self emails are redacted),
the role, whether it arrived via a team, and the history scope.

### Leaving a shared skill

If a skill was shared with you, you can drop your own direct grant. Owners cannot
leave their own skill.

- **CLI:** `goodeye skills leave <id-or-name>`
- **MCP tool:** `leave_shared_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/leave`

## Transferring ownership

Transferring a skill does not apply immediately. It creates an invitation
envelope; the recipient must accept it before ownership and the verifier
dependencies move over. A self-transfer is a no-op.

- **CLI:** `goodeye skills transfer-ownership <id-or-name> <new-owner>`
- **MCP tool:** `transfer_skill_ownership`
- **REST:** `POST /v1/skills/{id_or_slug}/transfer-ownership`

The recipient accepts with `goodeye invitations accept <id>` (or the
`accept_invitation` tool / `POST /v1/invitations/{id}/accept`). See [Teams](teams.md)
for the full invitation flow.

## Fork lineage

A skill created by forking a [template](templates.md) carries lineage back to the
exact template version it came from. Lineage reports the parent template, the
pinned version, the upstream latest version, and whether the upstream was later
archived, had the pinned version deprecated, or was permanently deleted. A fork
keeps its own content copy: if the parent template is permanently deleted, the
fork still works and lineage reports the source as permanently deleted.

- **CLI:** `goodeye skills lineage <id-or-name>` (also
  `goodeye templates lineage <id-or-name>`, same view)
- **MCP tool:** `lookup_fork_lineage`
- **REST:** `GET /v1/skills/{id_or_slug}/lineage`

## See also

- [Verifiers](verifiers.md): deploy the semantic checks a skill references.
- [Auditing skills](auditing-skills.md): grade a skill against the authoring
  checks.
- [Templates](templates.md): publish a skill publicly, fork one back, and manage
  the public lifecycle.
- [Teams](teams.md): grants, teams, and the invitation flow.
- [CLI](cli.md), [MCP](mcp.md), [REST API](rest-api.md): the three surfaces in
  detail.
- [Accounts and Billing](accounts-and-billing.md): handles, API keys, and usage.
