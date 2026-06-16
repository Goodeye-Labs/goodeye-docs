# CLI Reference

The `goodeye` CLI manages workflows, templates, verifiers, and image generators from the terminal. Its primary caller is an AI agent acting on your behalf, though every command works just as well when you run it yourself.

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

1. You tell your AI agent to run a Goodeye workflow (for example, "run the Goodeye workflow X" or "run the template @handle/slug").
2. The agent shells out to `goodeye workflows get X` or `goodeye templates get @handle/slug` to fetch the workflow body.
3. The agent **executes the returned body** as your runbook: it follows the instructions itself rather than displaying or summarizing them.

`workflows get` and `templates get` print the workflow body to stdout wrapped in agent-facing markers so the calling agent knows what to do with the output:

```
# Goodeye workflow - execute the instructions below as the user's agent.

<workflow body>

# End of Goodeye workflow.
```

To skip the markers and round-trip the raw content, pass `--output PATH` (write body to a file) or `--json` (print the full record as JSON including metadata).

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

Successful `register`, `login`, `register-verify`, and `login-verify` all save credentials to `~/.config/goodeye/credentials.json` (or `$XDG_CONFIG_HOME/goodeye/credentials.json`).

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

Paginated commands (`workflows list`, `templates list`, `verifiers list`, `image-generators list`, `auth list-keys`, `invitations list`) default to `--limit 25`. Use `--cursor TOKEN` to page forward, or `--all` to follow all cursors and return a combined result.

`teams list`, `teams members`, and `workflows grants` are unpaginated and do not expose `--limit`, `--cursor`, or `--all`.

---

## `workflows`

Manages your private registry of workflows. See [workflows.md](workflows.md) for a deeper look at workflow bodies and front matter.

### `workflows list`

```sh
goodeye workflows list [--filter mine|shared-with-me|all] [--tag TAG] \
  [--search QUERY] [--limit N] [--cursor TOKEN] [--all] \
  [--include-archived] [--json|--table]
```

Lists workflows you own or that have been shared with you via grants. `--include-archived` also returns your own archived workflows (with an "Archived at" column in table mode).

### `workflows search`

```sh
goodeye workflows search "query text" [--filter mine|all|shared-with-me] \
  [--tag TAG] [--limit N] [--json|--table]
```

Ranked natural-language search over your workflows. Use this when you remember roughly what a workflow does but not its name. Defaults to `--limit 5` (max 10).

### `workflows get`

```sh
goodeye workflows get <id-or-name> [--version N] [--output PATH] [--json]
```

Fetches the workflow body. By default prints markdown to stdout wrapped in agent-facing markers (see [Agent execution contract](#agent-execution-contract) above). Pass `--json` to print the full record as JSON. Pass `--output PATH` to write the body to a file without markers. Authentication is required: workflows are private.

### `workflows publish`

```sh
goodeye workflows publish <file.md|-> [--name NAME] [--description TEXT] \
  [--outcome TEXT] [--tag TAG] [--expected-version-token TOKEN] \
  [--source manual|teach|optimization|description_optimization|audit] [--verifier NAME=UUID[@VERSION]] \
  [--clear-verifiers] [--image-generator NAME=REF[@VERSION]] [--clear-image-generators] [--clear-files]
```

Saves a workflow from a markdown file, a directory containing `SKILL.md`, or stdin (use `-` for stdin, which is preferred for generated output). Metadata comes from command-line flags, YAML front matter in the markdown, or both: flags override front matter. `name`, `description`, and `outcome` are required. Repeat `--tag` to attach multiple tags.

Front matter format:

```markdown
---
name: my-workflow
description: One sentence on what this workflow does and when to use it.
outcome: Reduce refund-row mislabels.
tags: [data, cleanup]
---
```

If a workflow with the same name already exists under your account, a new version is appended. Pass `--expected-version-token TOKEN` (from the previous response or `workflows list`) to confirm the parent version and prevent accidental overwrites.

Workflows are always private. To share one publicly, run `goodeye templates publish` as a separate step.

### `workflows archive` / `workflows unarchive`

```sh
goodeye workflows archive <id-or-name> [--yes]   # reversible hide
goodeye workflows unarchive <id-or-name>           # restore
```

Archiving hides the workflow from list results and grants but keeps all versions and files intact. Use `archive` instead of `delete` when you want a reversible alternative.

### `workflows delete` / `workflows delete-version`

```sh
goodeye workflows delete <id-or-name> [--yes]
goodeye workflows delete-version <id-or-name> <version> [--yes]
```

**Permanent and immediate.** `delete` removes the workflow, all its versions, all attached files, and all access grants. `delete-version` removes a single non-current version and its files. There is no recovery path through any product surface.

### `workflows grant` / `workflows revoke-grant` / `workflows grants` / `workflows leave`

```sh
goodeye workflows grant <id-or-name> <grantee> view|edit|admin [--include-history]
goodeye workflows revoke-grant <id-or-name> <grantee>
goodeye workflows grants <id-or-name> [--json|--table]
goodeye workflows leave <id-or-name> [--yes]
```

Share a workflow with a user (by email or handle) or a team (by handle). By default the grantee sees the version current at share time and later. Pass `--include-history` to share the full version history. `grants` lists all current grants. `leave` removes your own direct grant from a workflow someone else shared with you.

### `workflows teach`

```sh
goodeye workflows teach <id-or-name>
```

Fetches the teach pack for an existing workflow and prints it to stdout for the calling agent to follow. Use to refine an existing workflow through an interactive teach session, then persist the result with `workflows publish ... --source teach`.

### `workflows optimize`

```sh
goodeye workflows optimize <id-or-name> [--max-iterations N]
```

Fetches the optimize pack for an existing workflow (defaults to 20 iterations, max 1000). The pack drives an optimization loop; the caller saves the result explicitly with `workflows publish ... --source optimization` after user approval.

### `workflows optimize-description`

```sh
goodeye workflows optimize-description <id-or-name> [--max-iterations N]
```

Fetches the description-optimize pack for an existing workflow (defaults to 10 iterations, max 1000). The pack tunes the workflow's `description`, the text that decides when it fires, for trigger accuracy. It changes only the description; the caller saves the result explicitly with `workflows publish ... --source description_optimization` after user approval.

### `workflows audit`

```sh
goodeye workflows audit [<id-or-name>]
```

Fetches the audit pack for a workflow, or for a local skill when you omit the id. The pack inspects the workflow against a best-practice rubric and returns a priority-ranked report (P0, P1, P2) with a concrete fix for each finding, enforcing at least one criterion with a platform LLM judge. You apply only the fixes you approve; they are saved explicitly with `workflows publish ... --source audit`, editing a local copy first when one exists.

### `workflows check-safety`

```sh
goodeye workflows check-safety <id-or-name[@N]> [--version N] [--json]
```

Runs safety checks on a workflow version. Returns `clean`, `flagged`, or `blocked`. Each call consumes two metered verifier runs.

### `workflows transfer-ownership`

```sh
goodeye workflows transfer-ownership <id-or-name> <new-owner>
```

Initiates an ownership transfer. Returns an invitation the recipient must accept with `goodeye invitations accept <id>`.

### `workflows lineage`

```sh
goodeye workflows lineage <id-or-name> [--json]
```

Shows a workflow's fork lineage: the parent template, pinned version, and whether the source was archived, permanently deleted, or had its version deprecated.

### `workflows sync`

```sh
goodeye workflows sync [--target DIR] [--force] [--yes] [--json|--table]
```

Pulls every configured sync target and then reports status. Equivalent to `sync pull` followed by `sync status`. Subcommands:

| Subcommand | Purpose |
|---|---|
| `sync target add <DIR>` | Configure a local directory to mirror workflows into. Pass `--preset claude`, `--preset agents`, or `--preset cursor` instead of a path for known locations. |
| `sync target list` | List configured sync targets. |
| `sync target remove <DIR>` | Remove a configured sync target. |
| `sync pull [SLUG...]` | Pull workflows from the registry to local directories. |
| `sync push [SLUG...]` | Upload locally edited workflows back to the registry. |
| `sync status` | Report drift between the registry and local directories without writing anything. |
| `sync auto on [--interval <seconds>]` | Enable the automatic background pull (opt-in; default interval 3600 s). |
| `sync auto off` | Disable the automatic background pull. |
| `sync auto` | Print the current auto-pull setting and last run time. |

**`--scope`** on `sync target add` controls which workflows land in that directory: `owned` (default), `all` (owned plus shared), or `selected` (only slugs or globs supplied with `--only`).

### `workflows sync auto`

```sh
goodeye workflows sync auto on [--interval <seconds>]
goodeye workflows sync auto off
goodeye workflows sync auto
```

Manages the opt-in automatic background pull. When enabled, the CLI pulls the
safe set (new and updated workflows) in the background after your command
completes. It never overwrites local edits, never deletes local directories, and
never pushes.

- `on` enables auto-pull. Pass `--interval <seconds>` to set the minimum gap
  between automatic pulls (default: 3600, which is one hour).
- `off` disables it.
- With no subcommand, prints the current setting (on/off, interval, last run
  time).

The automatic pull applies to all configured sync targets and is suppressed in
CI environments, for `--json` output, for `--help`/`--version` and the bare
invocation, and while you are running a `workflows sync` command yourself, so it
never shadows a manual sync. Workflows with local edits or conflicts
are reported but never clobbered; workflows deleted on the registry are reported
but never removed locally. See [Syncing a bundle locally](workflows.md#syncing-a-bundle-locally)
for the full behavior.

---

## `templates`

Manages the public template catalog. Templates are the public form of a workflow, addressable as `@handle/slug`. See [templates.md](templates.md) for the full publishing lifecycle.

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

Fetches a public template by UUID, `@handle/slug`, or `@handle/slug@vN`. No authentication required. Non-owner reads include an unverified-template safety banner in the body. The output format and flags behave identically to `workflows get` (agent-facing markers by default, `--json` for the full record, `--output PATH` to write to a file).

**Tip:** if a handle was renamed and the old URL now redirects, the CLI prints a note to stderr so downstream processes that captured stdout are not affected.

### `templates publish`

```sh
goodeye templates publish <workflow-uuid-or-name> [--release-notes TEXT]
```

Publishes a private workflow as a new public template version. The first publish creates the template (slug is reused from the workflow); subsequent calls append a monotonic version. Requires a claimed handle (run `goodeye me claim-handle` first). Every publish runs automated safety checks. If the block verifier fails, the command exits with code 2 and does not publish.

### `templates fork`

```sh
goodeye templates fork <identifier> [--version N] [--name NAME]
```

Copies a public template into your private workflow namespace. Authentication required. If the forked version carries a deprecation warning, the CLI prints it to stderr. To fetch the body and run it, follow with `goodeye workflows get <workflow-id>`.

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

Runs safety checks on a public template version. Pass `--anonymous` to run without credentials. Returns `clean`, `flagged`, or `blocked`.

### `templates transfer-ownership`

```sh
goodeye templates transfer-ownership <template-ref> <new-owner>
```

Initiates an ownership transfer. Returns an invitation the recipient must accept.

### `templates lineage`

```sh
goodeye templates lineage <workflow-ref> [--json]
```

Shows a forked workflow's lineage relative to the template it was forked from. Pass the forked workflow's id, not a template id. Equivalent to `workflows lineage`.

---

## `verifiers`

Manages owner-scoped LLM judges. Each verifier scores a single criterion ("does this output satisfy this rule?") and returns pass/fail plus reasoning. Workflows reference deployed verifiers by UUID or `UUID@version`. See [verifiers.md](verifiers.md) for a deeper look at criterion writing and calibration examples.

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
  [--workflow-id UUID] [--workflow-version N] [--workflow-ref TEXT] \
  [--run-id TEXT] [--anonymous] [--json]
```

Runs a verifier and prints `PASS` or `FAIL` plus reasoning. `<verifier_id>` accepts a UUID, `system:<name>`, or your caller-owned name (the name is resolved to a UUID before the call).

- `--inputs-json`: JSON object whose keys must match the deployed `input_fields` exactly.
- `--media-url`: required for `text_image` and `image` contracts.
- `--workflow-id`, `--workflow-version`, `--workflow-ref`, `--run-id`: optional provenance fields stamped onto the run row.
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

Manages owner-scoped image generation configurations. Workflows can reference a deployed generator by UUID to run image generation with consistent settings. See [image-generators.md](image-generators.md) for usage patterns.

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
  [--workflow-id UUID] [--workflow-version N] [--workflow-ref TEXT] \
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

## `teams`

Manages teams and membership. Teams can be used as grantees for shared workflows. See [teams.md](teams.md) for team-based access patterns.

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

Invitations are created by `teams add-member`, `teams transfer-ownership`, `workflows transfer-ownership`, and `templates transfer-ownership`.

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

## `design`

```sh
goodeye design          # print the workflow-designer prompt to stdout
goodeye design --json   # print the full response object as JSON
```

Pipe the printed prompt into your AI assistant to start designing a workflow with built-in verifiers.

**Note:** the designer prompt is the recommended starting point for new workflows. After the design session, save the result with `goodeye workflows publish -`.

---

## Global options

| Flag | Description |
|---|---|
| `--version` | Show the CLI version and exit. |
| `--help` | Show help for any command or subcommand. |

---

## See also

- [Getting started](getting-started.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [Image generators](image-generators.md)
- [Teams](teams.md)
- [Accounts and billing](accounts-and-billing.md)
- [REST API reference](rest-api.md)
- [MCP integration](mcp.md)
