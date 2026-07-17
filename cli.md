# CLI Reference

The `goodeye` CLI manages skills, templates, verifiers, and image generators from the terminal. Its primary caller is an AI agent acting on your behalf, though every command works just as well when you run it yourself.

## Install

Requires Python 3.12 or later.

```sh
uv tool install goodeye
# or: pipx install goodeye
# or: pip install goodeye
```

Once installed, the `goodeye` command is available on your `PATH`.

### Update

```sh
goodeye update          # update to the latest PyPI release automatically
goodeye update --check  # check whether an update is available without installing
```

`goodeye update` detects how the CLI was installed and runs the appropriate upgrade command. If the install method cannot be auto-detected, the command prints the equivalent manual commands instead.

The CLI also checks PyPI silently in the background (at most every four hours) and may print a short notice to stderr when a newer release is available. Notices are suppressed in CI environments, for `--json` output, and for `goodeye update` itself, so machine-readable stdout stays clean.

---

## Agent execution contract

The primary caller of this CLI is an AI agent acting on your behalf, not a human at a prompt. The intended flow:

1. You tell your AI agent to run a Goodeye skill (for example, "run the Goodeye skill X" or "run the template @handle/slug").
2. The agent shells out to `goodeye skills get X` or `goodeye templates get @handle/slug` to fetch the skill body.
3. The agent **executes the returned body** as your runbook: it follows the instructions itself rather than displaying or summarizing them.

`skills get` and `templates get` stream the skill body to stdout. The body opens with a standing directive that tells the calling agent to run the skill on your behalf rather than display it, to say up front what the skill produces and what checks gate the result, to quote each verifier's real pass or fail and its reasoning, and to close with a short rundown of how the output was checked. That directive is what makes stdout the runbook path.

To round-trip the raw content instead, pass `--output PATH` (writes the directive-free body to a file, so it stays editable and re-publishable) or `--json` (prints the full record as JSON including metadata).

---

## Authentication

### Interactive sign-in (for humans)

```sh
goodeye register   # create a new account
goodeye login      # sign in to an existing account
```

Both open a device-code browser flow and save credentials locally on success. They share the same hosted sign-in page, which creates the account for new users and signs in returning users, so either command works whether or not you already have an account. On that page you can continue with your Google account or with email.

### Non-interactive email-code flow (for agents and automation)

Create a new account:

```sh
goodeye register --email you@example.com
goodeye register-verify --email you@example.com --code 123456
```

Sign in to an existing account:

```sh
goodeye login --email you@example.com
goodeye login-verify --email you@example.com --code 123456
```

Successful `register`, `login`, `register-verify`, and `login-verify` all save credentials to `~/.config/goodeye/credentials.json` (or `$XDG_CONFIG_HOME/goodeye/credentials.json`). All four accept an optional `--referral-code <code>` to redeem a referral bonus during sign-in.

### Sign out

```sh
goodeye logout
```

Removes the locally saved credentials. The underlying API key stays valid on the server. Use `goodeye auth revoke-key` to disable it.

### Check who you are

```sh
goodeye whoami           # show email and handle
goodeye whoami --json    # machine-readable output
```

### API keys (`good_live_` keys)

API keys let agents and scripts authenticate without an interactive sign-in. All commands accept a `good_live_` key via the `GOODEYE_API_KEY` environment variable or a stored credentials file.

```sh
# Create a key (the secret is shown once: save it)
goodeye auth create-key --name "my-agent-key"
goodeye auth create-key --name "my-agent-key" --copy   # also copy to clipboard

# List keys (secrets are never shown)
goodeye auth list-keys
goodeye auth list-keys --json

# Revoke a key (stops working immediately)
goodeye auth revoke-key <key-id-or-name>
```

Use the key as `Authorization: Bearer good_live_EXAMPLE_xxxxxxxx` for both the REST API and the MCP transport. See [accounts-and-billing.md](accounts-and-billing.md) for tier and credit details.

### Credential lookup order

1. `GOODEYE_API_KEY` environment variable.
2. `~/.config/goodeye/credentials.json` (or `$XDG_CONFIG_HOME/goodeye/credentials.json`).

### Server override

By default the CLI targets `https://api.goodeye.dev`. Override with the `GOODEYE_SERVER` environment variable or the `server` field in `credentials.json`.

---

## Output modes and pagination

List and search commands are TTY-aware: when stdout is a terminal they print a table by default; when stdout is redirected or piped they print compact single-line JSON. Pass `--table` or `--json` to choose explicitly (the flags are mutually exclusive).

JSON list output is wrapped in an object:
- Paginated lists: `{"items": [...], "next_cursor": "..."}` (or `null` when no more pages).
- Unpaginated lists: `{"items": [...]}`.
- Search commands: `{"items": [...], "query": "...", "limit": N, "search_mode": "..."}`.

Paginated commands (`skills list`, `templates list`, `verifiers list`, `image-generators list`, `images list`, `auth list-keys`) default to `--limit 25`. Use `--cursor TOKEN` to page forward, or `--all` to follow all cursors and return a combined result.

`invitations list` is also paginated (`--limit`, `--cursor`), but it uses the server's default page size and does not support `--all`.

`teams list`, `teams members`, and `skills grants` are unpaginated and do not expose `--limit`, `--cursor`, or `--all`.

---

## `skills`

Create, version, share, and run your private skills. See [skills.md](skills.md) for a deeper look at skill bodies and front matter.

### `skills list`

```sh
goodeye skills list [--filter mine|shared-with-me|all] [--tag TAG] \
  [--search QUERY] [--limit N] [--cursor TOKEN] [--all] \
  [--include-archived] [--json|--table]
```

Lists skills you own or that have been shared with you via grants. `--include-archived` also returns your own archived skills (with an "Archived at" column in table mode).

### `skills search`

```sh
goodeye skills search "query text" [--filter mine|all|shared-with-me] \
  [--tag TAG] [--limit N] [--json|--table]
```

Ranked natural-language search over your skills. Use this when you remember roughly what a skill does but not its name. Defaults to `--limit 5` (max 10).

### `skills get`

```sh
goodeye skills get <id-or-name> [--version N] [--output PATH] [--json]
```

Fetches the skill body. By default prints markdown to stdout, opening with the standing run directive (see [Agent execution contract](#agent-execution-contract) above). Pass `--json` to print the full record as JSON. Pass `--output PATH` to write the directive-free body to a file. Authentication is required: skills are private.

### `skills publish`

```sh
goodeye skills publish <file.md|-> [--name NAME] [--description TEXT] \
  [--outcome TEXT] [--tag TAG] [--expected-version-token TOKEN] \
  [--source manual|teach|optimization|description_optimization|audit] [--verifier NAME=UUID[@VERSION]] \
  [--clear-verifiers] [--image-generator NAME=REF[@VERSION]] [--clear-image-generators] [--clear-files]
```

Saves a skill from a markdown file, a directory containing `SKILL.md`, or stdin (use `-` for stdin, which is preferred for generated output). Metadata comes from command-line flags, YAML front matter in the markdown, or both: flags override front matter. `name` and `description` are required; `--tag` and `--outcome` are optional. Repeat `--tag` to attach multiple tags.

Front matter format:

```markdown
---
name: my-skill
description: One sentence on what this skill does and when to use it.
tags: [data, cleanup]
---
```

If a skill with the same name already exists under your account, a new version is appended. Pass `--expected-version-token TOKEN` (from the previous response or `skills list`) to confirm the parent version and prevent accidental overwrites.

Skills are always private. To share one publicly, run `goodeye templates publish` as a separate step.

### `skills archive` / `skills unarchive`

```sh
goodeye skills archive <id-or-name> [--yes]   # reversible hide
goodeye skills unarchive <id-or-name>           # restore
```

Archiving hides the skill from list results and grants but keeps all versions and files intact. Use `archive` instead of `delete` when you want a reversible alternative.

### `skills delete` / `skills delete-version`

```sh
goodeye skills delete <id-or-name> [--yes]
goodeye skills delete-version <id-or-name> <version> [--yes]
```

**Permanent and immediate.** `delete` removes the skill, all its versions, all attached files, and all access grants. `delete-version` removes a single non-current version and its files. There is no recovery path through any product surface.

### `skills grant` / `skills revoke-grant` / `skills grants` / `skills leave`

```sh
goodeye skills grant <id-or-name> <grantee> view|edit|admin [--include-history]
goodeye skills revoke-grant <id-or-name> <grantee>
goodeye skills grants <id-or-name> [--json|--table]
goodeye skills leave <id-or-name> [--yes]
```

Share a skill with a user (by email or handle) or a team (by handle). By default the grantee sees the version current at share time and later. Pass `--include-history` to share the full version history. `grants` lists all current grants. `leave` removes your own direct grant from a skill someone else shared with you.

### `skills teach`

```sh
goodeye skills teach <id-or-name>
```

Fetches the teach pack for an existing skill and prints it to stdout for the calling agent to follow. Use to refine an existing skill through an interactive teach session, then persist the result with `skills publish ... --source teach`.

### `skills optimize`

```sh
goodeye skills optimize <id-or-name> [--max-iterations N]
```

Fetches the optimize pack for an existing skill (defaults to 20 iterations, max 1000). The pack drives an optimization loop; the caller saves the result explicitly with `skills publish ... --source optimization` after user approval.

### `skills optimize-description`

```sh
goodeye skills optimize-description <id-or-name> [--max-iterations N]
```

Fetches the description-optimize pack for an existing skill (defaults to 10 iterations, max 1000). The pack tunes the skill's `description`, the text that decides when it fires, for trigger accuracy. It changes only the description; the caller saves the result explicitly with `skills publish ... --source description_optimization` after user approval.

### `skills audit`

```sh
goodeye skills audit [<id-or-name>]
```

Fetches the audit pack for a hosted skill, or for a skill file on disk when you omit the id. The pack inspects the skill against the authoring checks and returns a priority-ranked report (P0, P1, P2) with a concrete fix for each finding, enforcing at least one criterion with a platform LLM judge. You apply only the fixes you approve; they are saved explicitly with `skills publish ... --source audit`, editing a local copy first when one exists.

### `skills check-safety`

```sh
goodeye skills check-safety <id-or-name[@N]> [--version N] [--json]
```

Runs safety checks on a skill version. Returns `clean`, `flagged`, `blocked`, or `error`. Each call counts as two verifier runs against your credits.

### `skills transfer-ownership`

```sh
goodeye skills transfer-ownership <id-or-name> <new-owner>
```

Initiates an ownership transfer. Returns an invitation the recipient must accept with `goodeye invitations accept <id>`.

### `skills lineage`

```sh
goodeye skills lineage <id-or-name> [--json]
```

Shows a skill's fork lineage: the parent template, pinned version, and whether the source was archived, permanently deleted, or had its version deprecated.

### `skills sync`

```sh
goodeye skills sync [--target DIR] [--force] [--yes] [--json|--table]
```

Pulls every configured sync target and then reports status. Equivalent to `sync pull` followed by `sync status`. Subcommands:

| Subcommand | Purpose |
|---|---|
| `sync target add <DIR>` | Configure a local directory to mirror your hosted skills into. Pass `--preset claude`, `--preset agents`, or `--preset cursor` instead of a path for known locations. |
| `sync target list` | List configured sync targets. |
| `sync target remove <DIR>` | Remove a configured sync target. |
| `sync pull [SLUG...]` | Write your hosted skills down to skill directories on disk. |
| `sync push [SLUG...]` | Upload a skill file you edited on disk back to its hosted skill. |
| `sync status` | Report drift between your hosted skills and the skill files on disk without writing anything. |
| `sync auto on [--interval <seconds>]` | Enable the automatic background pull (opt-in; default interval 3600 s). |
| `sync auto off` | Disable the automatic background pull. |
| `sync auto` | Print the current auto-pull setting and last run time. |

**`--scope`** on `sync target add` controls which of your hosted skills land in that directory: `owned` (default), `all` (owned plus shared), or `selected` (only slugs or globs supplied with `--only`).

The opt-in `sync auto` background pull (turn on with `sync auto on`, default
interval one hour) pulls only new and updated hosted skills after a command
completes: it never overwrites your local edits, deletes skill files on disk, or
pushes, and a local conflict is reported rather than clobbered. It is suppressed
in CI, for machine-readable output, and during a manual sync. See
[Syncing a bundle locally](skills.md#syncing-a-bundle-locally) for depth.

---

## `templates`

Manages the public template catalog. Templates are the public form of a skill, addressable as `@handle/slug`. See [templates.md](templates.md) for the full publishing lifecycle.

### `templates list`

```sh
goodeye templates list [--filter all|mine] [--search QUERY] [--limit N] \
  [--cursor TOKEN] [--all] [--include-archived] [--json|--table]
```

Browses the public template catalog. No authentication required.

### `templates search`

```sh
goodeye templates search "query text" [--filter all|mine] [--limit N] [--json|--table]
```

Ranked natural-language search over public templates. Defaults to `--limit 5` (max 10).

### `templates get`

```sh
goodeye templates get <identifier> [--version N] [--output PATH] [--json]
```

Fetches a public template by UUID, `@handle/slug`, or `@handle/slug@vN`. No authentication required. Non-owner reads include an unverified-template safety banner in the body. The output format and flags behave identically to `skills get` (the standing run directive by default, `--json` for the full record, `--output PATH` to write to a file).

**Tip:** if a handle was renamed and the old URL now redirects, the CLI prints a note to stderr so downstream processes that captured stdout are not affected.

### `templates get-file`

```sh
goodeye templates get-file <identifier> <path> --output PATH [--sha256 HASH]
```

Writes one attached file from a template (for example a demo preview image) to a local path as raw bytes; nothing is printed to stdout. No authentication required. Pass `--sha256` to content-address the fetch so a republished or removed file no longer resolves at a stale address.

### `templates publish`

```sh
goodeye templates publish <skill-uuid-or-name> [--release-notes TEXT]
```

Publishes a private skill as a new public template version. The first publish creates the template (slug is reused from the skill); subsequent calls add the next version number. Requires a claimed handle (run `goodeye me claim-handle` first). Every publish runs automated safety checks. If the block verifier fails, the command exits with code 2 and does not publish.

### `templates fork`

```sh
goodeye templates fork <identifier> [--version N] [--name NAME]
```

Copies a public template into your private skill namespace. Authentication required. If the forked version carries a deprecation warning, the CLI prints it to stderr. To fetch the body and run it, follow with `goodeye skills get <skill-id>`.

### `templates unpublish`

```sh
goodeye templates unpublish <template-ref> <version>
```

Soft-unpublishes a single version. Existing forks pinned to that version continue to work. The catalog hides the template if no live version remains.

### `templates archive` / `templates unarchive`

```sh
goodeye templates archive <template-ref> [--reason TEXT] [--yes]
goodeye templates unarchive <template-ref>
```

Archiving hides the template from the public catalog but keeps all versions and fork lineage intact. Existing forks continue to work. Reversible.

### `templates delete` / `templates delete-version`

```sh
goodeye templates delete <template-ref> [--yes]
goodeye templates delete-version <template-ref> <version> [--yes]
```

**Permanent and immediate.** A template with at least one still-published version is refused (409). Unpublish the relevant version(s) or archive the template first. Forks keep their own content; their parent pointer is severed.

### `templates deprecate-version`

```sh
goodeye templates deprecate-version <template-ref> <version> --message TEXT
```

Flags a single version as deprecated with a message shown to anyone who forks that version. The version remains reachable.

### `templates check-safety`

```sh
goodeye templates check-safety <identifier[@N]> [--version N] [--anonymous] [--json]
```

Runs safety checks on a public template version. Pass `--anonymous` to run without credentials. Returns `clean`, `flagged`, `blocked`, or `error`. If a field is too long for the check, it is shortened for the scan and the output notes this.

### `templates transfer-ownership`

```sh
goodeye templates transfer-ownership <template-ref> <new-owner>
```

Initiates an ownership transfer. Returns an invitation the recipient must accept.

### `templates lineage`

```sh
goodeye templates lineage <skill-ref> [--json]
```

Shows a forked skill's lineage relative to the template it was forked from. Pass the forked skill's id, not a template id. Equivalent to `skills lineage`.

---

## `verifiers`

Manages owner-scoped LLM judges. Each verifier scores a single criterion ("does this output satisfy this rule?") and returns pass/fail plus reasoning. Skills reference deployed verifiers by UUID or `UUID@version`. See [verifiers.md](verifiers.md) for a deeper look at criterion writing and calibration examples.

All `verifiers` subcommands require authentication.

### `verifiers deploy`

```sh
goodeye verifiers deploy <config.json|->
```

Deploys a verifier or appends a new version from a JSON config file or stdin (use `-`). Required fields: `name`, `description`, `criterion`, `input_contract`. `input_fields` is required for `text` and `text_image` contracts. `few_shot_examples` and `model_settings` are optional.

On success prints `verifier_id`, `version`, and `version_token`. Save the token: it is required for the next re-deploy (pass as `expected_version_token` in the config).

### `verifiers list`

```sh
goodeye verifiers list [--limit N] [--cursor TOKEN] [--all] [--json|--table]
```

Lists active (non-revoked) verifiers you own with their current version and version token.

### `verifiers show`

```sh
goodeye verifiers show <verifier_id> [--version N] [--json]
```

Shows one verifier version: criterion, input contract, input fields, calibration examples, and a `config_hash` for drift detection. Owner-only.

### `verifiers run`

```sh
goodeye verifiers run <verifier_id> \
  [--inputs-json '{"field": "value"}'] \
  [--media-url URL] [--version N] \
  [--skill-id UUID] [--skill-version N] [--skill-ref TEXT] \
  [--run-id TEXT] [--anonymous] [--json]
```

Runs a verifier and prints `PASS` or `FAIL` plus reasoning. `<verifier_id>` accepts a UUID, `system:<name>`, or your caller-owned name (the name is resolved to a UUID before the call).

- `--inputs-json`: JSON object whose keys must match the deployed `input_fields` exactly.
- `--media-url`: required for `text_image` and `image` contracts.
- `--skill-id`, `--skill-version`, `--skill-ref`, `--run-id`: optional provenance fields stamped onto the run row.
- `--anonymous`: run without credentials, spending against a small per-IP credit budget. Requires a UUID or `system:<name>`.

The command exits 0 on a successful judgment regardless of pass/fail. Check the `PASS`/`FAIL` line or the `--json` output's `passed` field to gate downstream actions.

### `verifiers revoke`

```sh
goodeye verifiers revoke <verifier_id> [--yes]
```

Sets the verifier to `revoked` and removes it from list, show, and run. Irreversible. Existing run rows are retained for audit. Replace by deploying a fresh verifier under a new name.

### `verifiers delete`

```sh
goodeye verifiers delete <verifier_id> [--yes]
```

**Permanent and immediate.** Removes the verifier, all its versions, and all run records. A verifier referenced by a live published template version is refused (409). Unpublish the relevant template version(s) first.

---

## `image-generators`

Manages owner-scoped image generation configurations. Skills can reference a deployed generator by UUID to run image generation with consistent settings. See [image-generators.md](image-generators.md) for usage patterns.

Most subcommands require authentication. `generate` supports `--anonymous` for public preview.

### `image-generators deploy`

```sh
goodeye image-generators deploy \
  --name NAME --description TEXT \
  --provider fal --model PROVIDER/MODEL \
  --generation-contract text_to_image|image_to_image \
  [--params-json '{"key": "value"}'] \
  [--expected-version-token TOKEN]
```

Deploys an image generator or appends a new version. On success prints `generator_id`, `version`, and `version_token`. Save the token for re-deploys.

### `image-generators list`

```sh
goodeye image-generators list [--limit N] [--cursor TOKEN] [--all] [--json|--table]
```

Lists active (non-revoked) image generators you own.

### `image-generators show`

```sh
goodeye image-generators show <generator_id> [--version N] [--json]
```

Shows one generator version: model, contract, default parameters, and `config_hash`. Owner-only.

### `image-generators generate`

```sh
goodeye image-generators generate \
  --prompt "text description" \
  [--generator UUID|system:image-standard] \
  [--model PROVIDER/MODEL] \
  [--reference-image-url URL] \
  [--num-images 1-4] [--seed N] \
  [--params-json '{"key": "value"}'] \
  [--version N] \
  [--skill-id UUID] [--skill-version N] [--skill-ref TEXT] \
  [--run-id TEXT] [--anonymous] [--json]
```

Generates one or more images. Image URLs are printed to stdout (one per line) so the result can be piped. Cost and run metadata go to stderr.

`--generator` and `--model` are mutually exclusive. When neither is supplied, the platform standard quality tier (`system:image-standard`) is used. `--anonymous` requires a system tier or a UUID that appears in a published template.

### `image-generators revoke`

```sh
goodeye image-generators revoke <generator_id> [--yes]
```

Revokes a generator. Irreversible. Replace by deploying under a new name.

### `image-generators delete`

```sh
goodeye image-generators delete <generator_id> [--yes]
```

**Permanent and immediate.** A generator referenced by a live published template version is refused (409). Unpublish first.

---

## `images`

Uploads and manages hosted images with stable URLs (including images produced by a generator). Images are private by default; public images are reachable by URL without credentials. All subcommands require authentication. See [images.md](images.md) for usage patterns.

```sh
goodeye images upload <file> [--visibility public|private] [--ttl SECONDS] [--json]
goodeye images list [--source upload|generated] [--visibility public|private] \
  [--limit N] [--cursor TOKEN] [--all] [--json|--table]
goodeye images get <image-id> [--json]
goodeye images update <image-id> [--visibility public|private] \
  [--ttl SECONDS|--permanent] [--rotate-view-secret] [--json]
goodeye images delete <image-id> [--yes]
```

Uploads default to `private` and to no expiry; pass `--ttl SECONDS` to set a lifetime. A private image is reachable through a private view link you can forward; the plain URL stays locked.

Four shortcuts wrap `images update` for common edits:

| Shortcut | Effect |
|---|---|
| `images share <image-id>` | Make the image public. |
| `images unshare <image-id>` | Make the image private again. |
| `images set-ttl <image-id> <seconds\|permanent>` | Set a new expiry, or `permanent` to remove it. |
| `images reset-link <image-id>` | Issue a fresh private view link and revoke links shared earlier. |

---

## `teams`

Manages teams and membership. Teams can be used as grantees for shared skills. See [teams.md](teams.md) for team-based access patterns.

```sh
goodeye teams create <handle>
goodeye teams delete <team> [--yes]
goodeye teams list [--filter all|mine|member] [--json|--table]
goodeye teams members <team> [--json|--table]
goodeye teams add-member <team> <user>
goodeye teams remove-member <team> <user>
goodeye teams transfer-ownership <team> <new-owner>
```

`add-member` and `transfer-ownership` return invitations the recipient must accept. `remove-member` can be used by the team owner to remove anyone, or by a member to leave the team themselves.

---

## `invitations`

Invitations are created by `teams add-member`, `teams transfer-ownership`, `skills transfer-ownership`, and `templates transfer-ownership`.

```sh
goodeye invitations list [--filter received|sent|all] [--state pending|all] \
  [--limit N] [--cursor TOKEN] [--json|--table]
goodeye invitations accept <invitation-id>
goodeye invitations decline <invitation-id>
goodeye invitations cancel <invitation-id>
```

`cancel` is for senders. `accept` and `decline` are for recipients.

---

## `me`

View and update your profile. Handles are your public publish identity and are shared across users and teams.

```sh
goodeye me claim-handle <handle>    # claim a handle (3 to 40 chars, lowercase alphanumeric with hyphens)
goodeye me rename-handle <handle>   # change a claimed handle (subject to cooldown and yearly cap)
```

Old handle URLs for published templates redirect for a 90-day window after a rename.

---

## `usage`

```sh
goodeye usage           # show tier, available credit, and refill date
goodeye usage --json    # machine-readable output
```

Shows your current tier, available credits, monthly refill amount and date, and any unpaid balance. See [accounts-and-billing.md](accounts-and-billing.md) for tier details.

---

## `referrals`

```sh
goodeye referrals status [--json]   # show your code, redemptions, and credits earned
goodeye referrals redeem <code>     # redeem someone else's code for bonus credits
```

Both require authentication. You can also pass `--referral-code <code>` to `register` or `login` (and their `-verify` steps) to redeem during sign-in. See [referrals.md](referrals.md).

---

## `design`

```sh
goodeye design          # print the skill-designer prompt to stdout
goodeye design --json   # print the full response object as JSON
```

Pipe the printed prompt into your AI assistant to start designing a skill with built-in verifiers. Requires sign-in: the command errors without credentials.

**Note:** the designer prompt is the recommended starting point for new skills. After the design session, save the result with `goodeye skills publish -`.

---

## Global options

| Flag | Description |
|---|---|
| `--version` | Show the CLI version and exit. |
| `--help` | Show help for any command or subcommand. |

---

## See also

- [Getting started](getting-started.md)
- [Skills](skills.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [Image generators](image-generators.md)
- [Images](images.md)
- [Teams](teams.md)
- [Accounts and billing](accounts-and-billing.md)
- [Referrals](referrals.md)
- [REST API reference](rest-api.md)
- [MCP integration](mcp.md)
