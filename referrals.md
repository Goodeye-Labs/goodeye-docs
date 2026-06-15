# Referrals

Invite new users to Goodeye and both sides earn bonus credits. The program is two-sided with an asymmetric trigger: the new user receives bonus credits immediately when they redeem your code, and you (the inviter) receive bonus credits once that referred user activates their account.

## Your referral code

Every account has one shareable referral code. It is created the first time you view your referral status. Share it anywhere: a message, a post, a README, wherever new users might see it.

### Viewing your code and stats

```sh
goodeye referral status
```

```http
GET /v1/referrals/me
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `get_referral_status`

Example output:

```
Your referral code: K7MNP2QR
Instructions: Share this code anywhere. Anyone who redeems K7MNP2QR becomes your referral.
Redeemed: 8
Qualified: 3
Credits earned: $15.00
Slots remaining: 2
```

The response fields are:

| Field | Description |
|---|---|
| `code` | Your unique shareable referral code |
| `instructions` | A ready-to-paste snippet explaining how to redeem the code |
| `redeemed_count` | Total number of people who redeemed your code |
| `qualified_count` | Number of those who have activated and earned you a reward |
| `credits_earned_usd` | Total bonus credits you have earned from referrals |
| `slots_remaining` | How many more referral rewards you can earn |

## Sharing your code

Paste your code and a short note anywhere new users might see it: a Slack message, a social post, a README, an email. There is no built-in email-invite step. Anyone who redeems the code becomes your referral automatically.

## Redeeming a code

To claim the new-user bonus, redeem a referral code. You can do this as a standalone command, or as part of signing in or registering.

### Standalone redeem

```sh
goodeye referral redeem <code>
```

```http
POST /v1/referrals/redeem
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"code": "<code>"}
```

MCP tool: `redeem_referral_code` (connect your account first, then call it)

### Redeeming during sign-in or registration

Pass `--referral-code` to the sign-in or registration commands and the code is redeemed automatically right after you authenticate:

```sh
# Interactive sign-in (OAuth or email-code flow)
goodeye login --referral-code <code>

# Email-code registration: verify step
goodeye register-verify --email <email> --code <code> --referral-code <code>
```

If the code cannot be applied (for example, you have already redeemed one), the sign-in still completes and the CLI tells you why the code was not applied.

### Eligibility

A redemption is accepted when all of the following are true:

- You are signed in to a Goodeye account.
- You have not redeemed any referral code before (one lifetime redemption per account).
- Your account is not yet activated (see [What activation means](#what-activation-means) below).
- The code is valid and not revoked.
- The code does not belong to your own account.

## What activation means

Your bonus credits land as soon as you redeem a valid code. The inviter's reward is separate: it unlocks when you (the referred user) activate your account.

Activation means your account meets both of these conditions:

- You own at least one private workflow.
- You have run a verifier or generated an image.

Browsing, searching, or viewing templates does not count toward activation. There is no time limit on activation: a referral stays pending until the referred user activates, however long that takes.

## Bonus credits and expiry

Bonus credits appear in your available balance alongside your monthly grant and are spent the same way, with no special restrictions. They expire one year after they are granted.

The inviter's reward lands shortly after the referred user activates. A background process reconciles pending referrals, so it can take a few minutes to appear. Each inviter can earn rewards for a limited number of referrals (shown as "slots remaining" in `goodeye referral status`).

Run `goodeye referral status` to see your total credits earned from referrals and how many reward slots you have left. The exact bonus for a single redemption is shown to the redeemer when they redeem a code (as credits granted). Amounts are not hardcoded in this document because they may change.

## Errors

These errors can be returned by the redeem endpoint or the `redeem_referral_code` MCP tool:

| Error slug | Meaning |
|---|---|
| `referral_code_not_found` | The code does not match any referral code |
| `self_referral` | You cannot redeem your own code |
| `already_referred` | This account has already redeemed a referral code |
| `referral_not_eligible` | The redemption is not eligible (your account is already activated, the program is unavailable, or the code was revoked) |

## See also

- [Accounts and Billing](accounts-and-billing.md) for your credit balance and usage
- [Getting Started](getting-started.md) for signing in and first steps
