# Teams

Teams let you share skills with a whole group at once, so a new teammate can run
the skills they need the moment they join instead of waiting for a one-off grant.
You create a team with a handle, add members through invitations, and then grant a
skill to `@teamhandle`. Every current and future member gains the access level you
set on the grant, and the verifiers the skill references travel with it.

```diagram-grants
head: Skill owner | holds a private skill
A user | grant: view or edit
A team | grant: every member inherits the role
```

## How teams work

A team has three parts:

- A **handle**, chosen at creation and immutable afterward.
- An **owner**, the person who created it or last accepted an ownership transfer.
  The owner has implicit membership: they are not listed as a member internally
  but still appear in member listings.
- Zero or more **members**.

You need a claimed user handle before you can create a team. See
[Accounts and Billing](accounts-and-billing.md) for handle setup.

## Creating a team

The handle must be 3 to 40 characters, lowercase alphanumeric with hyphens, and
unique across the platform. It cannot be changed after creation. If your own user
handle is still provisional, creation fails with `handle_not_claimed`; run
`goodeye me claim-handle` first.

- **CLI:** `goodeye teams create my-team`
- **MCP tool:** `create_team`
- **REST:** `POST /v1/teams`

## Listing teams

The `filter` accepts `all` (default), `mine` (teams you own), or `member` (teams
you belong to). Each result carries `team_id`, `handle`, `owner_user_id`, your
`role`, and timestamps.

- **CLI:** `goodeye teams list` (`--filter mine|member|all`)
- **MCP tool:** `list_teams`
- **REST:** `GET /v1/teams` (`?filter=mine|member`)

## Deleting a team

Only the owner can delete a team. Deleting frees the team handle, which becomes
claimable again after a hold period (see
[Accounts and Billing](accounts-and-billing.md#handles)).

- **CLI:** `goodeye teams delete my-team` (`--yes` to skip the prompt)
- **MCP tool:** `delete_team`
- **REST:** `DELETE /v1/teams/{team_id_or_handle}`

## Managing members

### Listing members

The listing includes every member and the owner. Each row carries `user_id`,
`handle`, and `role`; your own email is shown, while other members' emails are
redacted and only their handle appears. Listing is visible to the owner and all
members; a non-member receives `404` so the team's existence stays masked.

- **CLI:** `goodeye teams members my-team`
- **MCP tool:** `list_team_members`
- **REST:** `GET /v1/teams/{team_id_or_handle}/members`

### Adding a member

Adding a member creates an invitation; the recipient must accept before they join
(see [Invitations](#invitations)). Identify them by UUID, email, or handle (with
or without the `@`). The response is an invitation envelope: tell the recipient to
run `goodeye invitations accept <invitation_id>`.

- **CLI:** `goodeye teams add-member my-team alice@example.com`
  (or `goodeye teams add-member my-team @alice`)
- **MCP tool:** `add_team_member`
- **REST:** `POST /v1/teams/{team_id_or_handle}/members`

### Removing a member

The owner can remove any member, and members can remove themselves. To remove the
owner, transfer ownership first.

- **CLI:** `goodeye teams remove-member my-team @alice`
- **MCP tool:** `remove_team_member`
- **REST:** `DELETE /v1/teams/{team_id_or_handle}/members/{user_identifier}`

## Transferring team ownership

Ownership transfer uses the invitation pattern: the current owner proposes the
transfer, and the recipient must accept before ownership changes hands. On accept,
the previous owner automatically becomes a member of the team.

- **CLI:** `goodeye teams transfer-ownership my-team @newowner`
- **MCP tool:** `transfer_team_ownership`
- **REST:** `POST /v1/teams/{team_id_or_handle}/transfer-ownership`

## Granting a skill to a team

Once your team exists, share a skill with the entire team in one step. All
current and future members can fetch that skill at the granted role. The
`grantee` is the team's `@handle`, and the role is one of `view`, `edit`, or
`admin`.

- **CLI:** `goodeye skills grant my-skill @my-team view`
- **MCP tool:** `grant_skill`
- **REST:** `POST /v1/skills/{id_or_slug}/grants`

The REST body names the grantee with `grantee_email_or_at_team_handle`:

```json
{"grantee_email_or_at_team_handle": "@my-team", "role": "view"}
```

For the full description of roles, version floors, and cascaded verifier grants,
see [Skills](skills.md#sharing-with-grants).

## Invitations

Every mutating team operation that affects another user goes through an
invitation: both adding a member and transferring ownership return an invitation
envelope that the recipient must explicitly accept. The recipient is also emailed
when the invitation is created, with a link to accept it.

When an operation creates a pending invitation, the response carries an
`invitation_id`, a `kind` (`team_membership` or `team_ownership`), and an
`expires_at`:

```json
{
  "invitation_id": "550e8400-e29b-41d4-a716-446655440000",
  "kind": "team_membership",
  "expires_at": "2026-07-11T00:00:00+00:00"
}
```

### Accepting or declining

Only the recipient can accept or decline. Accepting applies the underlying action
immediately (membership is created, or ownership changes hands); declining changes
nothing.

- **CLI:** `goodeye invitations accept <invitation_id>` /
  `goodeye invitations decline <invitation_id>`
- **MCP tool:** `accept_invitation` / `decline_invitation`
- **REST:** `POST /v1/invitations/{invitation_id}/accept` /
  `POST /v1/invitations/{invitation_id}/decline`

### Listing, cancelling, and expiry

List invitations with `--filter` (`received` default, `sent`, or `all`) and
`--state` (`pending` default, or `all`). Only the proposer can cancel a pending
invitation, which stops the recipient from accepting. Invitations expire
automatically after a set window; a later `add-member` or `transfer-ownership`
after expiry simply creates a fresh one.

- **CLI:** `goodeye invitations list` /
  `goodeye invitations cancel <invitation_id>`
- **MCP tool:** `list_invitations` / `cancel_invitation`
- **REST:** `GET /v1/invitations`

## See also

- [Skills](skills.md) for grant roles, version floors, and revoking access.
- [Accounts and Billing](accounts-and-billing.md) for handles, API keys, and
  usage.
- [Templates](templates.md) for public sharing via published template versions.
