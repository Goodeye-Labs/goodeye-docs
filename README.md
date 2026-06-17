# Goodeye Documentation

Official documentation for [Goodeye](https://goodeye.dev), an outcome-aligned AI
workflow registry. You author workflows as markdown runbooks tagged with the
business outcome they serve, pair them with verifiers that score an AI agent
against a measurable result, and share them publicly as templates. Everything is
reachable from three peer surfaces: the CLI, an MCP server, and a REST API.

## What is Goodeye?

Goodeye turns a stated business outcome into a deployed AI workflow. The chain
is: Outcome to KPI(s) to Task to Workflow plus Verifiers.

- **Workflow**: a markdown runbook stored privately in the registry, tagged with
  the outcome it serves. An agent fetches the body and executes it as a runbook.
- **Template**: the public form of a workflow, shared under your handle so other
  people and their agents can find, fetch, and fork it.
- **Verifier**: a check the workflow runs on agent output. Structural,
  functional, or semantic (an LLM judge calibrated with examples).
- **Image generator**: a deployed, owner-scoped image generation capability a
  workflow can call.
- **Hosted image**: an image stored by Goodeye with a stable URL that never
  changes, including images produced by a generator.

## Documentation

The rendered docs live at [goodeye.dev/docs](https://goodeye.dev/docs). This
repository is the canonical markdown source for that site and for AI coding
assistants via Context7. Pages:

| Page | Description |
|------|-------------|
| [Overview](overview.md) | Core concepts and the Goodeye mental model |
| [Getting Started](getting-started.md) | Install the CLI, sign in, fetch and run your first template |
| [CLI](cli.md) | Command reference for the `goodeye` CLI |
| [MCP](mcp.md) | Connect AI assistants via the Model Context Protocol |
| [REST API](rest-api.md) | Use Goodeye from the `/v1` REST API |
| [Workflows](workflows.md) | Author, version, teach, optimize, and share workflows |
| [Templates](templates.md) | Publish, fork, and manage public templates |
| [Verifiers](verifiers.md) | Score agent output with structural, functional, and semantic checks |
| [Image Generators](image-generators.md) | Deploy and run owner-scoped image generation |
| [Images](images.md) | Upload, host, and manage images; get durable URLs for generated images |
| [Teams](teams.md) | Collaborate with teams, grants, and invitations |
| [Accounts and Billing](accounts-and-billing.md) | Handles, API keys, usage, tiers, and credits |
| [Referrals](referrals.md) | Invite new users and earn bonus credits on both sides |

## Quick links

- **Product**: [goodeye.dev](https://goodeye.dev)
- **Rendered docs**: [goodeye.dev/docs](https://goodeye.dev/docs)
- **MCP server URL**: `https://mcp.goodeye.dev/mcp`
- **REST API base**: `https://api.goodeye.dev/v1`
- **CLI**: `pipx install goodeye` (source: [Goodeye-Labs/goodeye-cli](https://github.com/Goodeye-Labs/goodeye-cli))

## Using with AI assistants

Goodeye is built for AI agents acting on a user's behalf. When an agent fetches
a workflow or template body, it follows that body as the user's runbook: it
executes the instructions rather than summarizing them.

These docs are indexed by [Context7](https://context7.com) so AI coding
assistants can pull accurate, up-to-date Goodeye usage instructions instead of
guessing.

## License

The documentation in this repository is proprietary to Goodeye Labs and provided
under the [Goodeye Documentation License](LICENSE). You may read and use it to
work with Goodeye, allow AI and documentation tools such as Context7 to index
it, and freely reuse the code samples. Redistributing or creating derivative
documentation is not permitted. See [LICENSE](LICENSE) for details.
