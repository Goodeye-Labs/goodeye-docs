# REST API

The Goodeye REST API drives workflows, templates, verifiers, and image generators straight from your own application or automation pipeline.

## Base URL

```
https://api.goodeye.dev/v1
```

Interactive API docs are available at `https://api.goodeye.dev/v1/docs`.

## Authentication

Most endpoints require a `good_live_` API key. Pass it as a Bearer token:

```http
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

Create a key with `goodeye auth create-key` or `POST /v1/api-keys` after signing in. Keys are shown once at creation time; store them securely.

**Anonymous access:** the public template catalog (`GET /v1/templates`, `GET /v1/templates/{identifier}`, `POST /v1/templates/search`) is readable without authentication. Anonymous callers also have a small monthly credit grant for metered operations such as `POST /v1/verifiers/{id}/runs` against publicly referenced verifiers.

## Bootstrap and registration

Before a user has an API key, two endpoints let clients register and log in.

### `GET /.well-known/goodeye-client-config`

CLI bootstrap config (no auth required); programmatic clients should authenticate with an API key.

### Registration and login (email code flow)

```http
POST /v1/register
Content-Type: application/json

{"email": "you@example.com"}
```

```http
HTTP/1.1 202 Accepted
{"status": "ok", "message": "Check your email for next steps."}
```

Then verify the code from your inbox:

```http
POST /v1/register/verify
Content-Type: application/json

{"email": "you@example.com", "code": "123456"}
```

```json
{"api_key": "good_live_..."}
```

Use `POST /v1/login` and `POST /v1/login/verify` for returning accounts (same flow). The verify response includes your API key, which you can use for all subsequent requests.

### OAuth token exchange

`POST /v1/auth/exchange` is handled automatically by the CLI after an OAuth sign-in; direct API callers do not need it.

## Pagination

List endpoints return a cursor-paginated response:

```json
{
  "items": [...],
  "next_cursor": "<opaque-cursor>"
}
```

Pass `next_cursor` as the `cursor` query parameter on the next request. Treat the cursor as opaque: do not parse or construct it. When `next_cursor` is `null`, you are on the last page. Most list endpoints also accept a `limit` parameter.

## Error model

All errors return JSON with a stable `error` slug and a human-readable `message`. Some errors include a `hint` field with recovery guidance, and `retryable` when relevant.

```json
{
  "error": "not_found",
  "message": "workflow not found"
}
```

| HTTP status | Slug | When it occurs |
|-------------|------|----------------|
| 400 | `validation_error` | Bad request shape or parameter |
| 400 | `handle_invalid` | Handle format rejected |
| 401 | `auth_required` | Missing or invalid credentials |
| 401 | `invalid_credentials` | Bad email code at login/register |
| 402 | `budget_exhausted` | Monthly credit grant spent |
| 402 | `anonymous_daily_cap` | Shared daily limit on anonymous usage reached (anonymous callers only) |
| 403 | `account_suspended` | Account suspended |
| 403 | `forbidden` | Caller lacks the required role |
| 404 | `not_found` | Resource not found or masked (private resources are masked as not-found for unauthorized callers) |
| 404 | `user_not_found` | Target user does not exist |
| 409 | `conflict` | Duplicate, locked, or serving-gate refusal |
| 409 | `handle_taken` | Handle already claimed by another user |
| 409 | `handle_reserved` | Handle is reserved and cannot be claimed |
| 409 | `handle_already_claimed` | Caller already has a handle |
| 409 | `handle_not_claimed` | A handle is required before performing this action |
| 409 | `invitation_already_resolved` | Invitation was already accepted, declined, or cancelled |
| 409 | `invitation_already_pending` | An equivalent invitation is already pending |
| 410 | `invitation_expired` | Invitation has expired |
| 422 | `safety_verification_failed` | Template body failed the hard-block safety check at publish |
| 422 | `quota_exceeded` | A per-user quota was hit |
| 429 | `rate_limited` | Request rate limit exceeded |
| 429 | `handle_rename_too_soon` | Handle renamed within the 90-day cooldown |
| 429 | `handle_rename_limit_exceeded` | Handle rename calendar-year limit reached |
| 503 | `safety_verification_unavailable` | Safety verifier runtime unavailable at publish time; retry |
| 503 | `search_unavailable` | LLM-ranked search temporarily unavailable; use list with `search=` for lexical fallback |
| 503 | `verifier_unavailable` | Verifier LLM runtime unavailable; retry the run |

`safety_verification_failed` responses include `reasoning` (what to change) and `verifier_run_id` for audit.

## Account and profile

### `GET /v1/me`

Returns the authenticated caller's account id, email, and handle.

```http
GET /v1/me
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

```json
{
  "id": "9f8c1e2a-3b4d-4c5e-8f6a-1b2c3d4e5f60",
  "email": "you@example.com",
  "handle": "yourhandle",
  "handle_claimed_at": "2024-06-01T12:00:00Z"
}
```

### `PATCH /v1/me`

Claim a handle.

```http
PATCH /v1/me
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{"handle": "yourhandle"}
```

### `POST /v1/me/rename-handle`

Rename your claimed handle. Limited to once per rolling 90 days and 3 times per calendar year.

```http
POST /v1/me/rename-handle
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{"handle": "newhandle"}
```

### `GET /v1/me/usage`

Returns your current billing period usage: tier, period dates, granted/used/remaining credits, and unpaid balance.

```http
GET /v1/me/usage
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

See [Accounts and billing](accounts-and-billing.md) for tier details.

## API keys

### `POST /v1/api-keys`

Mint a new API key.

```http
POST /v1/api-keys
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{"name": "my-automation"}
```

```json
{
  "id": "key_...",
  "name": "my-automation",
  "key": "good_live_..."
}
```

**The `key` field is only returned once.** Store it immediately.

### `GET /v1/api-keys`

List your keys (metadata only, no secrets).

### `DELETE /v1/api-keys/{key_id}`

Revoke a key. Returns `204 No Content`. A second delete returns `404`.

## Referrals

```
GET  /v1/referrals/me        # your referral code, redemptions, and credits earned
POST /v1/referrals/redeem    # redeem another user's code for bonus credits
```

Redeem body: `{"code": "..."}`. Both require authentication. See [Referrals](referrals.md).

## Workflows

Workflows are private to their owner. Use [templates](templates.md) to share publicly.

### `POST /v1/workflows`

Create or update a workflow. Required fields: `name`, `body`. Optional: `description`, `outcome`, `tags`, `slug`, `files` (multi-file bundle).

```http
POST /v1/workflows
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "name": "Weekly report",
  "outcome": "Deliver a concise weekly summary to the team",
  "body": "# Weekly report\n\n...",
  "tags": ["reporting"]
}
```

```json
{
  "workflow_id": "89dcc843-d056-44d9-ae34-ebcff4903885",
  "version": 1,
  "version_token": "...",
  "name": "Weekly report",
  "slug": "weekly-report",
  "verifiers": []
}
```

### `GET /v1/workflows`

List workflows visible to you. Query params: `filter` (`mine`, `all`, `shared-with-me`), `tag`, `search`, `limit`, `cursor`, `include_archived`.

### `POST /v1/workflows/search`

Natural language search. Body: `{"query": "..."}`.

### `GET /v1/workflows/{id_or_slug}`

Fetch a workflow. Pass `?version=N` to pin a specific version. Add `Accept: text/markdown` to receive the body as plain markdown (the agent's runbook fetch path).

### `GET /v1/workflows/{id_or_slug}/versions/{version}`

Fetch a specific version directly.

### `GET /v1/workflows/{id_or_slug}/files`

Fetch files from a workflow bundle. Use `?path=relative/path.md` for a single file or `?paths=a.md&paths=b.md` for a batch. On a single-file fetch, `format=raw` returns the file's raw bytes instead of a JSON envelope, and `sha256=` content-addresses the fetch.

### `POST /v1/workflows/{id_or_slug}/archive`

Reversibly hide a workflow. The slug stays occupied and the workflow is recoverable.

### `POST /v1/workflows/{id_or_slug}/unarchive`

Restore an archived workflow.

### `DELETE /v1/workflows/{id_or_slug}`

Permanently erase a workflow and all its versions. No recovery. Works on live and archived workflows.

### `DELETE /v1/workflows/{id_or_slug}/versions/{n}`

Permanently erase a single non-current version. The current version cannot be erased this way; use `DELETE /v1/workflows/{id_or_slug}` to remove everything.

### `POST /v1/workflows/{id_or_slug}/teach`

Teach an existing workflow with new examples.

### `POST /v1/workflows/{id_or_slug}/optimize`

Run an iterative optimization loop. Optional query param `max_iterations` (1 to 1000).

### `POST /v1/workflows/{id_or_slug}/optimize-description`

Tune the workflow's trigger description for accuracy (description-only). Optional query param `max_iterations` (1 to 1000, defaults to 10).

### `POST /v1/workflows/{id_or_slug}/audit`

Audit a workflow against a best-practice rubric. Returns a priority-ranked report (P0, P1, P2) with concrete fixes and runs at least one platform LLM-judge criterion. Requires at least `view` access.

### `GET /v1/audit/workflow-prompt`

Fetch the audit pack with no workflow subject, for auditing a local skill that is not on Goodeye yet.

### `POST /v1/workflows/{id_or_slug}/safety-check`

Run both platform safety checks on a workflow version. Requires auth; bills two metered runs.

### Grants

```
POST   /v1/workflows/{id_or_slug}/grants      # grant access
GET    /v1/workflows/{id_or_slug}/grants      # list grants
DELETE /v1/workflows/{id_or_slug}/grants      # revoke a grant
POST   /v1/workflows/{id_or_slug}/leave       # leave a shared workflow
```

Grant body: `{"grantee_email_or_at_team_handle": "...", "role": "view|edit|admin", "include_history": false}`.

Revoke body: `{"grantee_email_or_at_team_handle": "..."}`.

### `POST /v1/workflows/{id_or_slug}/transfer-ownership`

Transfer ownership to another user. Returns an invitation envelope; the recipient calls `POST /v1/invitations/{id}/accept` to apply.

### `GET /v1/workflows/{id_or_slug}/lineage`

Trace a workflow back to its template fork source.

## Templates

Templates are the public form of a workflow. Anonymous callers can list, search, and read templates.

### `POST /v1/templates`

Publish a workflow as a new template version. Runs safety checks before any writes; returns `safety_verification` with `status` and optional `advisory_reasoning`.

Errors: `safety_verification_failed` (422) on a hard block, `safety_verification_unavailable` (503) if the runtime is unreachable.

```http
POST /v1/templates
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "workflow_id": "89dcc843-d056-44d9-ae34-ebcff4903885",
  "release_notes": "Initial release"
}
```

### `GET /v1/templates`

List public templates. Query params: `filter` (`all`, `mine`), `search`, `limit`, `cursor`, `include_archived` (only your own archived templates when authenticated).

### `POST /v1/templates/search`

Natural language search. Body: `{"query": "..."}`. Anonymous callers may search.

### `POST /v1/templates/fork`

Fork a template into a private workflow. Body: `{"identifier": "@handle/slug", "version": null, "name": null}`. Works for anonymous callers (creates an ephemeral workflow) or authenticated users (creates a persisted workflow).

### `GET /v1/templates/{identifier}`

Fetch a template. `identifier` is a UUID or `@handle/slug` (optionally `@handle/slug@vN`). Query param: `version`. Add `Accept: text/markdown` to receive the body directly.

**Note:** non-owners see a safety status banner prepended to the body. The banner is neutral: it shows the publishing handle and safety status only.

### `GET /v1/templates/{identifier}/files`

Fetch a single file from a template version's file tree. Query params: `path` (required), `format` (`raw` returns the file's raw bytes instead of a JSON envelope), `sha256` (content-address the fetch so a republished or removed file no longer resolves at a stale address).

### Version management

```
DELETE /v1/templates/{identifier}/versions/{v}             # unpublish a version
DELETE /v1/templates/{identifier}/versions/{v}/permanent   # permanently erase (must unpublish first)
POST   /v1/templates/{identifier}/versions/{v}/deprecate   # flag as deprecated without hiding
```

### `POST /v1/templates/{identifier}/safety-check`

Run platform safety checks on a template version. Auth optional; anonymous callers are billed against their anonymous credit grant. The scan reads the body, description, and outcome. If a field is too long to scan in full it is shortened for this check rather than failing: `truncated` is then `true` and `truncated_fields` names the shortened fields, so you know the verdict is partial. Invalid input returns `validation_error` (400).

### `POST /v1/templates/{identifier}/archive`

Reversibly hide from public listing. Optional body: `{"archive_reason": "..."}`.

### `POST /v1/templates/{identifier}/unarchive`

Restore an archived template.

### `DELETE /v1/templates/{identifier}`

Permanently erase a template and all its versions. Requires the template to be archived or to have no published versions. Forks keep their content; their parent pointer is set to null.

### `POST /v1/templates/{identifier}/transfer-ownership`

Transfer ownership to another user. Returns an invitation; the recipient must accept.

## Verifiers

### `POST /v1/verifiers`

Deploy a semantic verifier (create or new version). Required fields: `name`, `description`, `criterion`, `input_contract`. `input_fields` is required for `text` and `text_image` contracts; `few_shot_examples` and `model_settings` are optional.

```http
POST /v1/verifiers
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "name": "response-clarity",
  "description": "Scores whether an answer is clear, direct, and free of hedging.",
  "criterion": "Return passed=true when the response is clear, direct, and free of unnecessary qualifiers.",
  "input_contract": "text",
  "input_fields": ["user_query", "agent_response"],
  "few_shot_examples": [
    {
      "inputs": {"user_query": "What is 2+2?", "agent_response": "4"},
      "passed": true,
      "reasoning": "Direct and complete."
    }
  ]
}
```

```json
{
  "verifier_id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "name": "response-clarity",
  "version": 1,
  "version_token": "...",
  "status": "active"
}
```

Subsequent deploys must include `expected_version_token` (the token from the last deploy, list, or get). Token mismatch returns `409 conflict`.

### `GET /v1/verifiers`

List your active verifiers. Query params: `limit`, `cursor`.

### `GET /v1/verifiers/{verifier_id}`

Get a verifier version including criterion, calibration examples, and contract. Pass `?version=N` to pin. Accepts UUID or accessible user-scope verifier name.

### `POST /v1/verifiers/{verifier_id}/runs`

Execute a verifier judgment.

```http
POST /v1/verifiers/1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d/runs
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "inputs": {
    "user_query": "What is 2+2?",
    "agent_response": "It depends on your perspective."
  }
}
```

```json
{
  "verifier_run_id": "7f3e9c1b-2d4a-4e8f-9a6b-5c0d1e2f3a4b",
  "verifier_id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "version": 1,
  "status": "complete",
  "passed": false,
  "reasoning": "The response hedges a straightforward factual question.",
  "duration_ms": 342,
  "created_at": "2024-06-01T12:00:00Z"
}
```

Use `system:<name>` as the path segment to invoke a platform-managed verifier: `POST /v1/verifiers/system:workflow-design-qa/runs`. Anonymous callers may run a verifier UUID if it appears in a live public template snapshot.

### `DELETE /v1/verifiers/{verifier_id}`

Revoke a verifier (deactivate, keep run history).

### `DELETE /v1/verifiers/{verifier_id}/permanent`

Permanently erase a verifier and all its data. Refused if a live published template snapshot references it; unpublish the template version first.

## Image generators

### `POST /v1/image-generators`

Deploy an image generator.

### `GET /v1/image-generators`

List your active image generators. Query params: `limit`, `cursor`.

### `GET /v1/image-generators/{generator_id}`

Get an image generator version.

### `POST /v1/image-generators/{generator_id}/runs`

Generate images.

```http
POST /v1/image-generators/{generator_id}/runs
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: application/json

{
  "prompt": "A calm mountain lake at dawn, photorealistic",
  "num_images": 1
}
```

`generator_id` can be a UUID, `uuid@version` to pin a version, or `system:<tier>` for a platform quality tier. The `model` body field lets authenticated callers run a one-off generation without a deployed generator (use any placeholder path segment such as `-`).

### `DELETE /v1/image-generators/{generator_id}`

Revoke an image generator.

### `DELETE /v1/image-generators/{generator_id}/permanent`

Permanently erase an image generator and all its run records.

## Images

Hosted images with stable URLs (including images produced by a generator). Images are private by default; public images serve to anyone with the URL. Management routes require authentication; the byte-serve route is public.

```
POST   /v1/images                 # upload (multipart: file, visibility, ttl_seconds); returns 201
GET    /v1/images                 # list your images (filters: source, visibility; limit, cursor)
GET    /v1/images/{image_id}      # fetch one image record
PATCH  /v1/images/{image_id}      # update visibility, expiry, or view link
DELETE /v1/images/{image_id}      # permanently delete; returns 204
GET    /v1/i/{token}.{ext}        # public byte-serve (also HEAD); raw image bytes by token
```

Upload accepts PNG, JPEG, WebP, and GIF. `PATCH` body fields: `visibility`, `ttl_seconds`, `permanent` (mutually exclusive with `ttl_seconds`; clears the expiry), and `rotate_view_secret` (issue a fresh private view link and revoke links shared earlier). On `/v1/i/{token}.{ext}`, the extension is cosmetic; private images require the owner or a valid view-link secret.

Image-specific errors:

| HTTP status | Slug | When it occurs |
|-------------|------|----------------|
| 413 | `file_too_large` | Image file exceeds the maximum allowed size |
| 413 | `image_dimensions_exceeded` | Image resolution exceeds the maximum allowed pixels |
| 415 | `unsupported_image_type` | File is not a PNG, JPEG, WebP, or GIF |
| 422 | `image_content_rejected` | A public image was screened as disallowed content |
| 503 | `image_screening_unavailable` | Content screen unavailable; the image was not made public (retry) |

## Teams

All team endpoints require authentication.

```
POST   /v1/teams                                   # create a team
GET    /v1/teams                                   # list teams (filter: mine|member|all)
DELETE /v1/teams/{team_id}                         # delete a team
GET    /v1/teams/{team_id}/members                 # list members
POST   /v1/teams/{team_id}/members                 # add a member (returns invitation)
DELETE /v1/teams/{team_id}/members/{user_id}       # remove a member
POST   /v1/teams/{team_id}/transfer-ownership      # transfer ownership (returns invitation)
```

Create body: `{"handle": "my-team"}`.

Add member body: `{"user_identifier": "email@example.com"}`.

## Invitations

Transfers of workflow or template ownership, team membership additions, and team ownership transfers all go through invitations. The recipient must accept for the action to take effect.

```
GET    /v1/invitations                         # list (filter: received|sent|all; state: pending|all)
POST   /v1/invitations/{id}/accept             # accept
POST   /v1/invitations/{id}/decline            # decline
POST   /v1/invitations/{id}/cancel             # cancel (proposer only)
```

## Design

```
GET /v1/design/workflow-prompt
```

Returns the workflow designer prompt pack (the same payload the MCP `design_workflow` tool returns). Auth required.

## Health and observability

```
GET /healthz
```

Returns `{"status": "ok"}`. Available on every host without auth.

## See also

- [Getting started](getting-started.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [Image generators](image-generators.md)
- [Images](images.md)
- [Teams](teams.md)
- [Referrals](referrals.md)
- [MCP](mcp.md)
- [CLI](cli.md)
- [Accounts and billing](accounts-and-billing.md)
