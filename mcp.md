# MCP Integration

Connect an MCP-compatible AI assistant to Goodeye and it can manage your skills, publish templates, run verifiers, and generate images from natural language.

## Overview

Everything Goodeye does is reachable over MCP. Once connected, your AI assistant can:

- Design, save, fetch, and manage skills
- Publish, search, fork, and verify templates
- Deploy and run semantic verifiers
- Deploy and invoke image generators
- Upload and manage hosted images
- Manage teams, invitations, and grants
- Mint and revoke API keys
- Check your current usage and credits
- View and redeem referral codes

**Agent contract:** when your assistant calls `get_skill` or `get_template` and receives a body back, it executes that body as your runbook. It follows the instructions itself rather than summarizing or just displaying them.

## The endpoint

```
https://mcp.goodeye.dev/mcp
```

## Authentication

There are two ways in.

**OAuth (sign-in flow):** connect in your MCP client and it prompts you to sign in. On the hosted sign-in page you can continue with your Google account or with email, and your client is authorized once you approve. Use this for interactive assistants like Claude.ai, Cursor, and VS Code.

**API key:** create a `good_live_` key with `goodeye auth create-key` (or `POST /v1/api-keys`), then pass it as a `Bearer` token in your MCP client config. Use this for automated agents and CI pipelines.

```http
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
```

Anonymous callers cannot use the MCP server. Public template catalog browsing is available over the REST API without auth (see [REST API](rest-api.md)).

## Client setup

### Claude Code

Add the server with one command:

```sh
claude mcp add --transport http goodeye \
  https://mcp.goodeye.dev/mcp
```

This adds Goodeye to your user-level config, available in all projects. To scope it to a specific project instead, add `--scope project` before the server name.

To connect with an API key instead of OAuth:

```sh
claude mcp add --transport http goodeye \
  https://mcp.goodeye.dev/mcp \
  --header "Authorization: Bearer good_live_EXAMPLE_xxxxxxxx"
```

### Claude.ai and Claude Desktop

Claude.ai and Claude Desktop share the same connectors, so you only need to set this up once.

1. Go to [**Customize > Connectors**](https://claude.ai/settings/connectors?modal=add-custom-connector)
2. Click **Add custom connector**
3. Enter:
   - **Name:** Goodeye
   - **URL:** `https://mcp.goodeye.dev/mcp`
4. Click **Add**
5. When prompted, sign in with your Goodeye account to authorize access
6. Enable the connector in any conversation via the **+** button

To connect with an API key on Claude Desktop, you need a bridge tool. Add this to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "goodeye": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.goodeye.dev/mcp",
        "--header",
        "Authorization:${GOODEYE_API_KEY}"
      ],
      "env": {
        "GOODEYE_API_KEY": "Bearer good_live_EXAMPLE_xxxxxxxx"
      }
    }
  }
}
```

Replace `good_live_EXAMPLE_xxxxxxxx` with your actual API key and restart Claude.

### Cursor

Add Goodeye manually to your project's `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "goodeye": {
      "url": "https://mcp.goodeye.dev/mcp"
    }
  }
}
```

Restart Cursor after adding the config, then sign in when prompted.

To use an API key instead:

```json
{
  "mcpServers": {
    "goodeye": {
      "url": "https://mcp.goodeye.dev/mcp",
      "headers": {
        "Authorization": "Bearer good_live_EXAMPLE_xxxxxxxx"
      }
    }
  }
}
```

### VS Code

1. Open VS Code and press **Cmd/Ctrl + Shift + P**
2. Run **MCP: Add Server**
3. Select **HTTP** as the server type
4. Enter the URL: `https://mcp.goodeye.dev/mcp`
5. Give it the name `goodeye`

Or add it directly to `.vscode/settings.json`:

```json
{
  "mcp": {
    "servers": {
      "goodeye": {
        "url": "https://mcp.goodeye.dev/mcp"
      }
    }
  }
}
```

To use an API key:

```json
{
  "mcp": {
    "servers": {
      "goodeye": {
        "url": "https://mcp.goodeye.dev/mcp",
        "headers": {
          "Authorization": "Bearer good_live_EXAMPLE_xxxxxxxx"
        }
      }
    }
  }
}
```

Requires VS Code 1.99+ with GitHub Copilot enabled.

### Windsurf

1. Open Windsurf and press **Cmd/Ctrl + ,** to open Settings
2. Search for **MCP** or navigate to **Cascade** > **MCP Servers** > **View raw config**
3. Add the Goodeye server to the `mcpServers` object:

```json
{
  "mcpServers": {
    "goodeye": {
      "serverUrl": "https://mcp.goodeye.dev/mcp",
      "disabled": false
    }
  }
}
```

4. Save and click **Refresh** (or restart Windsurf)
5. Sign in with your Goodeye account when prompted

To use an API key:

```json
{
  "mcpServers": {
    "goodeye": {
      "serverUrl": "https://mcp.goodeye.dev/mcp",
      "headers": {
        "Authorization": "Bearer good_live_EXAMPLE_xxxxxxxx"
      },
      "disabled": false
    }
  }
}
```

## Available tools

Any authenticated caller gets every tool. The groups below follow the main resource areas.

### Skills

| Tool | What it does |
|------|--------------|
| `design_skill` | Start a guided skill and verifier design session. Returns the skill designer prompt pack. |
| `save_skill` | Create or update a skill. Accepts multi-file bundles. |
| `list_skills` | List skills you own or have been granted access to. |
| `search_skills` | Natural language search across your skills. |
| `get_skill` | Fetch a skill by id or slug. The agent executes the returned body as a runbook. |
| `get_skill_file` | Fetch a single file from a skill's file bundle. |
| `get_skill_files` | Fetch multiple files from a skill bundle in one call. |
| `archive_skill` | Reversibly hide a skill (slug stays occupied). |
| `unarchive_skill` | Restore an archived skill. |
| `delete_skill` | Permanently erase a skill. No recovery. |
| `delete_skill_version` | Permanently erase a single non-current skill version. |
| `teach_skill` | Teach an existing skill with new examples. |
| `optimize_skill` | Run an agent-driven iteration loop to improve a skill. |
| `optimize_description` | Tune a skill's trigger description for accuracy (description-only). |
| `audit_skill` | Audit a hosted skill, or a skill file on disk, against the authoring checks; returns a priority-ranked report with concrete fixes. |
| `check_skill_safety` | Run platform safety checks on a skill version. |
| `grant_skill` | Share a skill with another user or team. |
| `revoke_skill_grant` | Remove an access grant. |
| `list_skill_grants` | List who has access to a skill. |
| `leave_shared_skill` | Leave a skill someone else shared with you. |
| `transfer_skill_ownership` | Transfer ownership to another user (returns an invitation). |
| `lookup_fork_lineage` | Trace a skill back to its template source. |

### Templates

| Tool | What it does |
|------|--------------|
| `publish_template_version` | Publish a skill as a new public template version. Runs safety checks first. |
| `unpublish_template_version` | Hide a published template version. |
| `deprecate_template_version` | Flag a version as deprecated without hiding it. |
| `delete_template_version` | Permanently erase an unpublished template version. |
| `delete_template` | Permanently erase a template and all its versions. |
| `archive_template` | Reversibly hide a template from public listing. |
| `unarchive_template` | Restore an archived template. |
| `list_templates` | List public templates. |
| `search_templates` | Natural language search across public templates. |
| `get_template` | Fetch a template by UUID or `@handle/slug`. The agent executes the returned body as a runbook. |
| `get_template_file` | Fetch a single file from a template version's file tree. |
| `fork_template` | Copy a public template into a private skill. |
| `check_template_safety` | Run platform safety checks on a published template version. |
| `transfer_template_ownership` | Transfer template ownership to another user. |

### Verifiers

| Tool | What it does |
|------|--------------|
| `deploy_verifier` | Create or update a semantic verifier. |
| `list_verifiers` | List your verifiers. |
| `get_verifier` | Fetch a verifier version including criterion and calibration. |
| `run_verifier` | Execute a verifier judgment. Accepts UUID or `system:<name>` for platform verifiers. |
| `revoke_verifier` | Deactivate a verifier without erasing run history. |
| `delete_verifier` | Permanently erase a verifier and all its data. |

**Note:** platform-managed `system` verifiers are run-only via `system:<name>` aliases. They do not appear in `list_verifiers` or `get_verifier`, and their configuration is not exposed.

### Image generators

| Tool | What it does |
|------|--------------|
| `deploy_image_generator` | Create or update an image generator. |
| `list_image_generators` | List your image generators. |
| `get_image_generator` | Fetch an image generator version. |
| `generate_image` | Run an image generation call. |
| `revoke_image_generator` | Deactivate an image generator. |
| `delete_image_generator` | Permanently erase an image generator and all its run records. |

### Images

| Tool | What it does |
|------|--------------|
| `upload_image` | Upload an image and get a hosted URL. |
| `list_images` | List your hosted images. |
| `get_image` | Fetch one hosted image. |
| `update_image` | Change visibility, expiry, or view link. |
| `delete_image` | Permanently delete a hosted image. |

### Teams and invitations

| Tool | What it does |
|------|--------------|
| `create_team` | Create a team with a handle. |
| `list_teams` | List teams you own or belong to. |
| `delete_team` | Delete a team you own. |
| `list_team_members` | List members of a team. |
| `add_team_member` | Invite a user to a team (returns an invitation). |
| `remove_team_member` | Remove a member from a team. |
| `transfer_team_ownership` | Transfer team ownership (returns an invitation). |
| `list_invitations` | List pending invitations sent to or by you. |
| `accept_invitation` | Accept a pending invitation. |
| `decline_invitation` | Decline a pending invitation. |
| `cancel_invitation` | Cancel an invitation you sent. |

### Account

| Tool | What it does |
|------|--------------|
| `claim_handle` | Claim a public handle for publishing templates. |
| `rename_handle` | Rename your claimed handle (limited to once per 90 days, 3 per calendar year). |
| `mint_api_key` | Create a new API key. |
| `list_api_keys` | List your API keys (metadata only, secrets never returned again). |
| `revoke_api_key` | Revoke an API key. |
| `get_usage` | Get your current-period usage summary (granted, used, remaining). |
| `get_referral_status` | Get your referral code, redemptions, and credits earned. |
| `redeem_referral_code` | Redeem another user's referral code for bonus credits. |

## Troubleshooting

### "Authentication required" or 401 errors

- Try disconnecting and reconnecting Goodeye in your client settings
- Make sure you are signing in with the same account that owns your skills and keys
- If using an API key, verify the key is active and the `Authorization` header is formatted correctly: `Bearer good_live_...`

### Tools not appearing

- Restart your MCP client after adding the configuration
- Verify the endpoint URL is exactly `https://mcp.goodeye.dev/mcp`
- In Windsurf, make sure the server is not set to `"disabled": true`
- In VS Code, verify GitHub Copilot agent mode is enabled (requires VS Code 1.99+)
- Check your MCP client's logs for connection errors

### Budget or account errors

- `budget_exhausted` (402): your monthly credit grant is spent. Run `goodeye usage` or `GET /v1/me/usage` to check your balance. See [Accounts and billing](accounts-and-billing.md).
- `account_suspended` (403): contact [hello@goodeyelabs.com](mailto:hello@goodeyelabs.com)

## See also

- [Getting started](getting-started.md)
- [Skills](skills.md)
- [Templates](templates.md)
- [Verifiers](verifiers.md)
- [Images](images.md)
- [Referrals](referrals.md)
- [REST API](rest-api.md)
- [CLI](cli.md)
- [Accounts and billing](accounts-and-billing.md)
