# Accounts and Billing

This page covers your Goodeye identity (handles), API keys for programmatic access, and the credit model that gates LLM-powered features such as verifier runs, workflow optimization, and template safety checks.

## Handles

A handle is your public identity on the Goodeye platform. It is used as the author namespace when you publish templates (for example, `@alice/my-template`). Handles are lowercase alphanumeric with hyphens, 3 to 40 characters, start and end with an alphanumeric character, and are unique across all users and teams.

### Claiming your handle

After signing up, your account has a provisional (unclaimed) handle. You must claim a handle before you can create a team or publish a template.

```sh
goodeye me claim-handle alice
```

```http
PATCH /v1/me
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"handle": "alice"}
```

MCP tool: `claim_handle`

On success the response includes `handle` and `claimed_at`. Structured errors:

| Error slug | Meaning |
|---|---|
| `handle_invalid` | Format does not match the allowed pattern |
| `handle_reserved` | The name is reserved by the platform |
| `handle_taken` | Another user or team already holds this handle |
| `handle_already_claimed` | You already have a claimed handle (use `rename-handle` to change it) |

### Renaming your handle

```sh
goodeye me rename-handle newname
```

```http
POST /v1/me/rename-handle
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"handle": "newname"}
```

MCP tool: `rename_handle`

**Rate limits:** you may rename once per rolling 90-day window and at most three times per UTC calendar year. If you rename and then rename back (or reclaim a handle you released), and your old handle is still inside its 90-day reservation window, the reclaim is free and does not count against either limit.

Your old handle enters a 90-day reservation window so that any URLs you published under it continue to resolve via redirect. If your old handle was ever stamped on a published template version it is permanently reserved and cannot be claimed by anyone else.

Additional structured errors for rename:

| Error slug | HTTP status | Meaning |
|---|---|---|
| `handle_not_claimed` | 400 | You have not yet claimed a handle |
| `handle_rename_too_soon` | 429 | Less than 90 days have passed since your last rename |
| `handle_rename_limit_exceeded` | 429 | You have used all three renames for this UTC calendar year |

## API keys

API keys let you authenticate programmatically with both the REST API and the MCP endpoint. All keys are prefixed `good_live_`.

### Creating a key

```sh
goodeye auth create-key --name "CI pipeline"
goodeye auth create-key --name "MCP client" --copy   # also copies to clipboard
```

```http
POST /v1/api-keys
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"name": "CI pipeline"}
```

MCP tool: `mint_api_key`

The response includes `id`, `name`, `key`, and `created_at`. The `key` value is the full `good_live_...` secret and is shown exactly once. Store it immediately in a secrets manager or environment variable. It cannot be retrieved again.

### Listing keys

```sh
goodeye auth list-keys
goodeye auth list-keys --table
```

```http
GET /v1/api-keys
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `list_api_keys`

List responses include `id`, `name`, and `created_at` only. Secret material is never returned after creation.

### Revoking a key

```sh
goodeye auth revoke-key key_01ABC...
```

```http
DELETE /v1/api-keys/{key_id_or_name}
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `revoke_api_key`

A revoked key stops working immediately. Revocation cannot be undone. The REST route returns `204` on success and `404` on a second delete or an unknown key.

### Using a key

Pass your key in the `Authorization` header for both REST and MCP requests:

```http
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

**Security tips:**
- Treat API keys like passwords: store them in environment variables or a secrets manager, never in source code.
- Rotate keys regularly by creating a new one and revoking the old one.
- Revoke a key immediately if it is exposed.

## Usage and credits

Goodeye uses a credit system to meter LLM-powered features. Each account has a monthly grant that refills at the start of each billing period.

### Checking your usage

```sh
goodeye usage
goodeye usage --json
```

```http
GET /v1/me/usage
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `get_usage`

Example output:

```
Tier: hobby
Available: $3.21
  refills to $5.00 on 07/01/2026
```

The response fields are:

| Field | Description |
|---|---|
| `tier` | Your current tier (`hobby` or `pro`) |
| `available_usd` | Total spendable balance right now |
| `monthly_remaining_usd` | Remaining from your current monthly grant |
| `monthly_refill_usd` | Amount your monthly grant refills to |
| `monthly_refill_at` | ISO timestamp of your next refill |
| `referral_remaining_usd` | Remaining from referral bonus credits |
| `unpaid_balance_usd` | Overspend that reduces your next refill |

Run `goodeye usage` to see the actual dollar amounts for your tier. Amounts are not hardcoded in this document because they may change.

### Tiers

| Tier | Description |
|---|---|
| `hobby` | Default tier for new accounts. Includes a monthly credit grant for personal and exploratory use. |
| `pro` | Higher monthly credit grant, suitable for production workflows. Contact us to upgrade. |

Anonymous callers (no auth) who reach public REST endpoints that consume credits draw on a small shared monthly allowance. That is what lets someone run a verifier against a published template without signing in. Like the authenticated tiers, the allowance pays for Goodeye-metered work (verifier runs and template safety checks), not the model usage an agent incurs while executing a workflow body; your agent bills that through whatever model you run it on. When the shared anonymous allowance is used up, those calls return `402 anonymous_daily_cap` until it refreshes. Signing in gives you your own credits.

### Billing errors

| HTTP status | Error slug | Meaning |
|---|---|---|
| `402` | `budget_exhausted` | Your available credit balance is zero. Wait for your next monthly refill. |
| `402` | `anonymous_daily_cap` | The shared allowance for anonymous usage has been reached. Sign in for your own credits, or try again later. |
| `403` | `account_suspended` | Your account has been suspended. Contact support. |

When your budget is exhausted, LLM-powered operations (verifier runs, workflow optimization, design sessions) return `402`. Other operations (saving workflows, listing templates, managing teams) are not credit-gated and continue to work.

## Limits

Metered features are gated by your credit balance, not by fixed request counts: when the balance reaches zero, credit-gated operations return `402 budget_exhausted` and resume at your next refill, and anonymous usage is bounded by a shared allowance that returns `402 anonymous_daily_cap` when used up. The one fixed-count limit is on handle renames (see [Renaming your handle](#renaming-your-handle)).

## Versioning and deprecation

When a shipped surface (an MCP tool, a REST route, or a CLI command) changes in a way that would break existing callers, the old behavior keeps working for at least one release and the change is announced: REST responses carry `Deprecation` and `Sunset` headers, MCP tool responses include a `deprecation_warning`, and the CLI prints a deprecation line to stderr. Purely additive changes (a new field, a new optional flag, a new command) ship without ceremony.

## See also

- [Teams](teams.md) for sharing workflows with groups
- [Workflows](workflows.md) for saving and granting access to workflows
- [Verifiers](verifiers.md) for deploying and running semantic checks
- [Templates](templates.md) for publishing your workflows publicly
