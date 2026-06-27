# Referrals

Invite new users to Goodeye and both sides earn bonus credits. The program is two-sided with an asymmetric trigger: the new user receives bonus credits immediately when they redeem your code, and you (the inviter) receive bonus credits once that referred user activates their account.

## Your referral code

Every account has one shareable referral code, created the first time you view your referral status. Paste it and a short note anywhere new users might see it: a Slack message, a social post, a README, an email. There is no built-in email-invite step; anyone who redeems the code becomes your referral automatically.

### Viewing your code and stats

```sh
goodeye referrals status
```

```http
GET /v1/referrals/me
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `get_referral_status`

Example output:

```
Your referral code: K7MNP2QR

Instructions:
  Share this code and ask the new user to install the Goodeye CLI, then run
  `goodeye login --referral-code K7MNP2QR` to sign in and claim their bonus
  credits. If they use Goodeye through an MCP client, they can connect their
  account and redeem the code with the redeem referral tool.

Redeemed: 8
Activated: 3
Credits earned: $15.00
Slots remaining: 2
```

The response fields are:

| Field | Description |
|---|---|
| `code` | Your unique shareable referral code |
| `instructions` | A ready-to-paste snippet explaining how to redeem the code |
| `redeemed_count` | Total number of people who redeemed your code |
| `activated_count` | Number of people you referred who have activated their account (and earned you a reward, up to your slot limit) |
| `credits_earned_usd` | Total bonus credits you have earned from referrals |
| `slots_remaining` | How many more referral rewards you can earn |

## Redeeming a code

To claim the new-user bonus, redeem a referral code. You can do this as a standalone command, or as part of signing in or registering.

### Standalone redeem

```sh
goodeye referrals redeem <code>
```

```http
POST /v1/referrals/redeem
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"code": "<code>"}
```

MCP tool: `redeem_referral_code` (connect your account first, then call it)

### Redeeming during sign-in or registration

Pass `--referral-code` on `register`, `login`, or the verify step, and the code is redeemed automatically once your credentials are saved:

```sh
goodeye register --referral-code <code>
goodeye login --referral-code <code>
goodeye login-verify --email <email> --code <code> --referral-code <code>
goodeye register-verify --email <email> --code <code> --referral-code <code>
```

The flag is accepted on `register`, `login`, `login-verify`, and `register-verify`. The two-step `goodeye login --email` flow does not redeem on the first step (it just sends the email and exits), so pass `--referral-code` on the following `login-verify` step instead. If the code cannot be applied (for example, you have already redeemed one), the sign-in still completes and the CLI tells you why.

### Eligibility

A redemption is accepted when all of the following are true:

- You are signed in to a Goodeye account.
- You have not redeemed any referral code before (one lifetime redemption per account).
- Your account is not yet activated (see [What activation means](#what-activation-means) below).
- The code is valid and not revoked.
- The code does not belong to your own account.

## What activation means

Your bonus credits land as soon as you redeem a valid code. The inviter's reward is separate: it unlocks once you (the referred user) activate your account, which means both of these are true:

- You own at least one private workflow.
- You have run a verifier or generated an image.

There is no time limit: a referral stays pending until the referred user activates, however long that takes.

## Bonus credits and expiry

Bonus credits appear in your available balance alongside your monthly grant and are spent the same way, with no special restrictions. They expire one year after they are granted. See [Accounts and billing](accounts-and-billing.md) for how your balance works.

The inviter's reward lands shortly after the referred user activates; it can take a few minutes to appear. Each inviter can earn rewards for a limited number of referrals, shown as "slots remaining" in `goodeye referrals status`. Exact amounts are not listed here because they may change: the redeemer sees the bonus granted when they redeem, and `goodeye referrals status` shows your total earned and slots left.

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
