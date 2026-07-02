# Accounts and Billing

This page covers your Goodeye identity (handles), API keys for programmatic
access, and the credit model that meters Goodeye's LLM-powered features such as
verifier runs and image generation.

## Handles

A handle is your public identity on Goodeye. It is the author namespace when you
publish templates (for example, `@alice/my-template`). Handles are lowercase
alphanumeric with hyphens, 3 to 40 characters, start and end with an alphanumeric
character, and are unique across all users and teams.

### Claiming your handle

After signing up your account has a provisional (unclaimed) handle. You must claim
a handle before you can create a team or publish a template. On success the
response includes `handle` and `claimed_at`.

- **CLI:** `goodeye me claim-handle alice`
- **MCP tool:** `claim_handle`
- **REST:** `PATCH /v1/me`

| Error slug | Meaning |
|---|---|
| `handle_invalid` | Format does not match the allowed pattern |
| `handle_reserved` | The name is reserved by the platform |
| `handle_taken` | Another user or team already holds this handle |
| `handle_already_claimed` | You already have a claimed handle (use rename instead) |

### Renaming your handle

- **CLI:** `goodeye me rename-handle newname`
- **MCP tool:** `rename_handle`
- **REST:** `POST /v1/me/rename-handle`

You may rename once per rolling 90-day window and at most three times per UTC
calendar year. If you rename and then rename back (or reclaim a handle you
released) while your old handle is still inside its 90-day reservation window, the
reclaim is free and does not count against either limit.

Your old handle enters a 90-day reservation window so that any URLs you published
under it keep resolving by redirect. If your old handle was ever stamped on a
published template version, it is permanently reserved and can never be claimed by
anyone else.

| Error slug | HTTP status | Meaning |
|---|---|---|
| `handle_not_claimed` | 400 | You have not yet claimed a handle |
| `handle_rename_too_soon` | 429 | Less than 90 days since your last rename |
| `handle_rename_limit_exceeded` | 429 | You have used all three renames this calendar year |

## API keys

API keys authenticate programmatically with both the REST API and the MCP
endpoint. Every key is prefixed `good_live_`.

### Creating a key

The response includes `id`, `name`, `key`, and `created_at`. The `key` value is
the full `good_live_...` secret and is shown exactly once: store it immediately in
a secrets manager or environment variable, because it cannot be retrieved again.

- **CLI:** `goodeye auth create-key --name "CI pipeline"` (`--copy` also copies it
  to the clipboard)
- **MCP tool:** `mint_api_key`
- **REST:** `POST /v1/api-keys`

### Listing keys

List responses include `id`, `name`, and `created_at` only; secret material is
never returned after creation.

- **CLI:** `goodeye auth list-keys` (`--table`)
- **MCP tool:** `list_api_keys`
- **REST:** `GET /v1/api-keys`

### Revoking a key

A revoked key stops working immediately, and revocation cannot be undone. The REST
route returns `204` on success and `404` on a second delete or an unknown key.

- **CLI:** `goodeye auth revoke-key <key-id-or-name>`
- **MCP tool:** `revoke_api_key`
- **REST:** `DELETE /v1/api-keys/{key_id_or_name}`

### Using a key

Pass your key as a bearer token for both REST and MCP requests:

```
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

Treat API keys like passwords: keep them in environment variables or a secrets
manager, never in source code; rotate by creating a new key and revoking the old
one; and revoke a key immediately if it is exposed.

## Usage and credits

Goodeye meters its LLM-powered features with credits. Each account has a monthly
grant that refills at the start of each billing period. Saving, fetching, and
browsing are free; only the work that calls a frontier model draws credits.

| Metered (draws credits) | Free |
|---|---|
| Verifier runs, including safety checks | Saving, fetching, and listing workflows and templates |
| Image generation, billed per image | Lexical `list` filtering and browsing |
| Natural-language (LLM-ranked) search | Designing, teaching, optimizing, and auditing a workflow |
| Public-image content screening | Managing teams, grants, handles, and API keys |

Designing, teaching, optimizing, and auditing return a prompt pack your agent runs
locally, so they draw no Goodeye credits. Your agent's own model usage while it
runs that prompt pack (or executes a workflow body) bills through whatever model
you run it on, separately from Goodeye.

### Checking your usage

- **CLI:** `goodeye usage` (`--json`)
- **MCP tool:** `get_usage`
- **REST:** `GET /v1/me/usage`

Example output:

```
Tier: hobby
Available: $3.21
  refills to $5.00 on 07/01/2026
```

| Field | Description |
|---|---|
| `tier` | Your current tier (`hobby` or `pro`) |
| `available_usd` | Total spendable balance right now |
| `monthly_remaining_usd` | Remaining from your current monthly grant |
| `monthly_refill_usd` | Amount your monthly grant refills to |
| `monthly_refill_at` | ISO timestamp of your next refill |
| `referral_remaining_usd` | Remaining from referral bonus credits |
| `unpaid_balance_usd` | Overspend that reduces your next refill |

Amounts are not hardcoded in this document because they may change; run
`goodeye usage` for the actual figures on your account.

If you are on Pro, `goodeye usage` also shows your subscription status and
the date your access renews, or, if you have cancelled, the date your Pro
access ends.

### Tiers

| Tier | Description |
|---|---|
| `hobby` | Default tier for new accounts. A monthly credit grant for personal and exploratory use. |
| `pro` | A paid subscription (20 USD per month, billed through Stripe) with a higher monthly credit grant than Hobby, suitable for production workflows. Upgrade yourself any time; see [Upgrading to Pro](#upgrading-to-pro). |

### Upgrading to Pro

Pro is a paid subscription: 20 USD per month, billed through Stripe, with a
higher monthly credit grant than Hobby.

- **CLI:** `goodeye subscription upgrade`
- **MCP tool:** `upgrade_to_pro`
- **REST:** `POST /v1/billing/checkout`

Any of these returns a secure Stripe-hosted checkout link. Open it and enter
your payment method once; Pro activates automatically as soon as the payment
clears. You can also just ask your AI agent to upgrade you to Pro: the same
capability is available over MCP and the REST API, so an agent acting on your
behalf can complete the upgrade without you leaving the conversation.

Unused Pro credit does not roll over. Each renewal replaces your Pro monthly
grant rather than adding to whatever was left over from the prior period.

### Cancelling or downgrading

- **CLI:** `goodeye subscription cancel` (alias: `goodeye downgrade`)
- **MCP tool:** `cancel_subscription`
- **REST:** `POST /v1/billing/subscription/cancel`

Cancellation takes effect at the end of your current paid period, not
immediately: you keep Pro access and your Pro credits until then, and your
account returns to Hobby once the period ends. There is no refund or
proration for the remaining time on a period you already paid for.

### Updating your card or viewing invoices

- **CLI:** `goodeye subscription portal`
- **MCP tool:** `create_billing_portal_session`
- **REST:** `POST /v1/billing/portal`

This opens the Stripe-hosted billing portal, where you can update your
payment method, view your invoice history, and recover from a failed
payment.

### Anonymous usage

Anonymous callers (no auth) can reach the public REST endpoints that consume
credits, which is what lets someone run a verifier against a published template
without signing in. Each anonymous caller gets its own small monthly grant, the
same way an authenticated account does; exhausting it returns `402
budget_exhausted`. Separately, total anonymous spend across everyone is bounded by
a shared daily ceiling that returns `402 anonymous_daily_cap` until the next day.
Either way the grant covers Goodeye-metered work (verifier runs and safety
checks), not the model usage your agent incurs while executing a workflow body.
Signing in gives you your own credits.

### Billing errors

| HTTP status | Error slug | Meaning |
|---|---|---|
| `402` | `budget_exhausted` | Your available credit balance is zero. Wait for your next monthly refill. |
| `402` | `anonymous_daily_cap` | The shared daily ceiling for all anonymous usage has been reached. Sign in for your own credits, or try again after the next day. |
| `403` | `account_suspended` | Your account has been suspended. [Contact us](mailto:hello@goodeyelabs.com). |
| `400` | `billing_not_enabled` | Self-service subscription billing is not enabled on this deployment. |
| `409` | `already_subscribed` | You already have an active Pro subscription. Manage it from the [billing portal](#updating-your-card-or-viewing-invoices) instead of starting a new checkout. |
| `409` | `no_active_subscription` | There is no active subscription to cancel. |

Metered features are gated by your credit balance, not by fixed request counts:
when the balance reaches zero those operations return `402` and resume at your
next refill, while other operations keep working. The one fixed-count limit is on
handle renames (see [Renaming your handle](#renaming-your-handle)).

## Versioning and deprecation

When a shipped surface (an MCP tool, a REST route, or a CLI command) changes in a
way that would break existing callers, the old behavior keeps working for at least
one release and the change is announced: REST responses carry `Deprecation` and
`Sunset` headers, MCP tool responses include a `deprecation_warning`, and the CLI
prints a deprecation line to stderr. Purely additive changes (a new field, a new
optional flag, a new command) ship without ceremony.

## See also

- [Teams](teams.md) for sharing workflows with groups.
- [Workflows](workflows.md) for saving and granting access to workflows.
- [Verifiers](verifiers.md) for deploying and running semantic checks.
- [Templates](templates.md) for publishing your workflows publicly.
