# Teams

Teams let you share workflows with a whole group at once, so a new teammate can run the workflows they need the moment they join instead of waiting for a one-off grant. You create a team with a handle, add members via invitations, and then grant workflows to `@teamhandle`. Every current and future member gains the access level you specified on the grant.

```diagram-grants
head: Workflow owner | holds a private workflow
A user | grant: view or edit
A team | grant: every member inherits the role
```

## How teams work

A team has:
- A **handle** (chosen at creation, immutable after that)
- An **owner** (the person who created it, or whoever last accepted an ownership transfer)
- Zero or more **members**

The owner always has implicit membership. They are not added as a member, but they still appear in `list-members` output.

You must have a claimed user handle before you can create a team. See [Accounts and billing](accounts-and-billing.md) for handle setup.

## Creating a team

```sh
goodeye teams create my-team
```

```http
POST /v1/teams
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"handle": "my-team"}
```

MCP tool: `create_team`

The handle must be 3 to 40 characters, lowercase alphanumeric with hyphens, and unique across the platform. It cannot be changed after creation.

**Note:** Team creation fails with `handle_not_claimed` if your user handle is still provisional. Run `goodeye me claim-handle` first.

## Listing teams

```sh
goodeye teams list
goodeye teams list --filter mine     # only teams you own
goodeye teams list --filter member   # only teams you are a member of
```

```http
GET /v1/teams
GET /v1/teams?filter=mine
GET /v1/teams?filter=member
```

MCP tool: `list_teams`

The `filter` parameter accepts `all` (default), `mine`, or `member`. Each item in the response includes `team_id`, `handle`, `owner_user_id`, `role`, `created_at`, and `updated_at`.

## Deleting a team

```sh
goodeye teams delete my-team
goodeye teams delete my-team --yes   # skip confirmation
```

```http
DELETE /v1/teams/{team_id_or_handle}
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `delete_team`

Only the owner can delete a team. Deleting frees the team handle, which becomes claimable again after a hold period (see [Accounts and billing](accounts-and-billing.md#handles)).

## Managing members

### Listing members

```sh
goodeye teams members my-team
```

```http
GET /v1/teams/{team_id_or_handle}/members
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `list_team_members`

The response lists every member including the owner. Each row includes `user_id`, `handle`, and `role`. Email is included only for your own row; other members' emails are redacted and the handle is shown instead.

Listing is visible to the owner and all members. Non-members receive `404` to preserve existence masking.

### Adding a member

Adding a member creates an invitation. The recipient must accept before they are added to the team.

```sh
goodeye teams add-member my-team alice@example.com
goodeye teams add-member my-team @alice
```

```http
POST /v1/teams/{team_id_or_handle}/members
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"user_identifier": "alice@example.com"}
```

MCP tool: `add_team_member`

You can identify the recipient by UUID, email address, or handle (with or without the `@` prefix). The response is an [invitation envelope](#invitations); tell the recipient to run `goodeye invitations accept <invitation_id>`. The membership is not active until they do.

### Removing a member

```sh
goodeye teams remove-member my-team @alice
```

```http
DELETE /v1/teams/{team_id_or_handle}/members/{user_identifier}
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tool: `remove_team_member`

The owner can remove any member. Members can remove themselves (self-leave). Passing the owner as `user_identifier` when you are the owner raises an error: you must transfer ownership first.

## Transferring team ownership

Ownership transfers use the invitation pattern: the current owner proposes the transfer, and the recipient must accept before ownership changes hands.

```sh
goodeye teams transfer-ownership my-team @newowner
```

```http
POST /v1/teams/{team_id_or_handle}/transfer-ownership
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"new_owner_user_identifier": "@newowner"}
```

MCP tool: `transfer_team_ownership`

On accept, the previous owner becomes a member of the team automatically.

## Granting a workflow to a team

Once your team exists, you can share a workflow with the entire team in one step:

```sh
goodeye workflows grant my-workflow @my-team --role view
```

```http
POST /v1/workflows/{id_or_slug}/grants
Content-Type: application/json
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx

{"grantee": "@my-team", "role": "view"}
```

MCP tool: `grant_workflow`

All current and future members of `@my-team` will be able to fetch that workflow at the granted role level. For a full description of roles, version floors, and cascaded verifier grants, see [Workflows](workflows.md).

## Invitations

Every mutating team operation that affects another user goes through an invitation: adding a member and transferring ownership both return an invitation envelope that the recipient must explicitly accept.

### The invitation envelope

When an operation creates a pending invitation, the response looks like this:

```json
{
  "invitation_id": "550e8400-e29b-41d4-a716-446655440000",
  "kind": "team_membership",
  "expires_at": "2026-07-11T00:00:00+00:00"
}
```

The `kind` field is `team_membership` or `team_ownership`.

### Accepting or declining

Only the recipient can accept or decline. Accepting applies the underlying action immediately (membership is created, or ownership changes hands); declining changes nothing.

```sh
goodeye invitations accept <invitation_id>
goodeye invitations decline <invitation_id>
```

```http
POST /v1/invitations/{invitation_id}/accept
POST /v1/invitations/{invitation_id}/decline
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

MCP tools: `accept_invitation`, `decline_invitation`.

### Listing, cancelling, and expiry

List invitations with `goodeye invitations list` (MCP `list_invitations`, REST `GET /v1/invitations`); `--filter` accepts `received` (default), `sent`, or `all`, and `--state` accepts `pending` (default) or `all`. Only the proposer can cancel a pending invitation (`goodeye invitations cancel <invitation_id>`, MCP `cancel_invitation`), which stops the recipient from accepting. Invitations expire automatically after a platform-defined window; a second `add-member` or `transfer-ownership` call after expiry creates a fresh one.

## See also

- [Workflows](workflows.md) for grant roles, version floors, and revoking access
- [Accounts and billing](accounts-and-billing.md) for handles, API keys, and usage
- [Templates](templates.md) for public sharing via published template versions
