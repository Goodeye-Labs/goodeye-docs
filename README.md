# Goodeye Documentation

Official documentation for [Goodeye](https://goodeye.dev), a private home for the
skills your AI follows and the verifiers its work must pass. You keep both in one
account, they stay private until you say otherwise, and they reach every machine
and every agent you run. Everything is reachable from three peer surfaces: the
CLI, an MCP server, and a REST API.

## What is Goodeye?

A skill is a markdown runbook. A verifier is a check the output has to clear.
Goodeye is where yours live.

Skills are private. Nothing is public until you publish a template, which is a
separate step you take on purpose. To share without going public, grant a named
user or team access, and the verifiers the skill references go with it at the
same version.

They also stay current everywhere you work. `goodeye skills sync` mirrors your
hosted skills into the directories your tools already read, so Claude Code,
Codex, Cursor, and anything else reading skill files from disk get the same
version. Automatic sync is on once you have a target, unless you turn it off, so
edit a skill once and every machine picks it up on its own.

Verifiers are hosted the same way. Deploy a semantic verifier once and every
skill that references it runs that exact version, on your laptop, in CI, or on
the machine of someone you granted it to.

The objects you work with:

- **Skill**: a markdown runbook stored privately in your Goodeye account. An agent
  fetches the body and executes it as a runbook.
- **Verifier**: a check the skill runs on agent output. Structural, functional,
  or semantic (an LLM judge calibrated with examples).
- **Grant**: private access to one of your skills for a named user or team.
- **Sync target**: a local directory Goodeye mirrors your hosted skills into.
- **Template**: the public form of a skill, shared under your handle so other
  people and their agents can find, fetch, and fork it.
- **Image generator**: a deployed, owner-scoped image generation capability a
  skill can call.
- **Hosted image**: an image stored by Goodeye with a stable URL that never
  changes, including images produced by a generator.

## Documentation

The rendered docs live at [goodeye.dev/docs](https://goodeye.dev/docs). This
repository is the canonical markdown source for that site and for AI coding
assistants via Context7. Pages:

| Page | Description |
|------|-------------|
| [Overview](overview.md) | Core concepts and the Goodeye mental model |
| [Getting Started](getting-started.md) | Install the CLI, host a skill you already have, sync it, and share it |
| [Skills](skills.md) | Author, version, teach, optimize, sync, and share skills |
| [Verifiers](verifiers.md) | Judge agent output with structural, functional, and semantic checks |
| [Templates](templates.md) | Publish, fork, and manage public templates |
| [Auditing skills](auditing-skills.md) | Grade a skill against the authoring checks |
| [Image Generators](image-generators.md) | Deploy and run owner-scoped image generation |
| [Images](images.md) | Upload, host, and manage images; get durable URLs for generated images |
| [Teams](teams.md) | Collaborate with teams, grants, and invitations |
| [Accounts and Billing](accounts-and-billing.md) | Handles, API keys, usage, tiers, and credits |
| [Referrals](referrals.md) | Invite new users and earn bonus credits on both sides |
| [CLI](cli.md) | Command reference for the `goodeye` CLI |
| [MCP](mcp.md) | Connect AI assistants via the Model Context Protocol |
| [REST API](rest-api.md) | Use Goodeye from the `/v1` REST API |

## Quick links

- **Product**: [goodeye.dev](https://goodeye.dev)
- **Rendered docs**: [goodeye.dev/docs](https://goodeye.dev/docs)
- **MCP server URL**: `https://mcp.goodeye.dev/mcp`
- **REST API base**: `https://api.goodeye.dev/v1`
- **CLI**: `pipx install goodeye` (source: [Goodeye-Labs/goodeye-cli](https://github.com/Goodeye-Labs/goodeye-cli))

## Using with AI assistants

Goodeye is built for AI agents acting on a user's behalf. When an agent fetches
a skill or template body, it follows that body as the user's runbook: it executes
the instructions rather than summarizing them.

These docs are indexed by [Context7](https://context7.com) so AI coding
assistants can pull accurate, up-to-date Goodeye usage instructions instead of
guessing.

## License

The documentation in this repository is proprietary to Goodeye Labs and provided
under the [Goodeye Documentation License](LICENSE). You may read and use it to
work with Goodeye, allow AI and documentation tools such as Context7 to index
it, and freely reuse the code samples. Redistributing or creating derivative
documentation is not permitted. See [LICENSE](LICENSE) for details.
