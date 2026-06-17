# Image hosting

Goodeye can host images for you and return a stable URL that never changes,
even if you regenerate, re-upload, or the originating session ends. This page
covers uploading images, managing visibility and expiry, and how generated
images now return durable hosted URLs automatically.

For the image generation capability itself, see
[Image generators](image-generators.md). For general workflow usage, see
[Workflows](workflows.md).

## Upload and host an image

You supply the image bytes and Goodeye stores them, returning a stable URL
you can embed or share. The URL does not change when you update the image's
metadata (visibility, TTL).

**CLI.**

```sh
goodeye images upload path/to/image.png \
  --visibility public \
  --ttl-seconds 604800
```

Flags:

- `--visibility public|private` (default: `private`): whether the URL is
  publicly readable without credentials.
- `--ttl-seconds <n>`: seconds until the image expires and stops resolving.
  Omit to keep the image until you delete it or set a TTL later.
- `--permanent`: store without any expiry (mutually exclusive with
  `--ttl-seconds`).
- `--json`: print the full response as JSON.

The command prints the `image_id` and the stable `image_url`.

**MCP tool.** `upload_image` accepts the image as a base64-encoded string
alongside `visibility`, `ttl_seconds`, and `permanent`.

**REST.**

```http
POST /v1/images
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: multipart/form-data

file=<binary>
visibility=public
ttl_seconds=604800
```

The response carries `{image_id, image_url, visibility, expires_at,
source, created_at}`.

## Public vs. private visibility

Every image starts as private unless you set `--visibility public` at upload
time. Private images require owner credentials to fetch; public images are
served to anyone with the URL and return cache headers so browsers and CDNs
can hold a copy.

You can flip visibility at any time without changing the URL:

```sh
# Make public (share)
goodeye images update <image_id> --visibility public

# Make private (revoke public access)
goodeye images update <image_id> --visibility private
```

The same update command works via MCP (`update_image`) and REST
(`PATCH /v1/images/{image_id}`).

## TTL and permanence

An image can have an expiry time (a TTL), be explicitly permanent, or simply
have no TTL set (it stays until you delete it).

- **Set a TTL**: the image stops resolving after the given number of seconds
  from the moment the TTL is applied (not from upload time).
- **Clear a TTL**: the image stays until you delete it or set a new TTL.
- **Mark permanent**: records a permanent intent; has the same effect as no
  TTL but makes intent explicit.
- `ttl_seconds` and `permanent` are mutually exclusive on any single call.

```sh
# Extend to 30 days from now
goodeye images update <image_id> --ttl-seconds 2592000

# Remove the expiry
goodeye images update <image_id> --clear-ttl

# Mark explicitly permanent
goodeye images update <image_id> --permanent
```

When an image expires, its URL stops resolving (404). The record is not
immediately purged; cleanup happens asynchronously.

## Durable URLs for generated images

When you call `generate_image` (or `goodeye image-generators generate`), each
output image is now automatically hosted and the response includes a durable
`hosted_image_url` alongside the provider URL. The hosted URL stays valid
after the provider session ends, so you can embed it in a workflow output, a
template, or any other surface without worrying about the link expiring.

The hosted copy inherits the visibility and TTL rules above. You can update
it with `goodeye images update` using the `image_id` returned in the
generation response.

## List, get, update, delete

All five management commands are available from the CLI, REST, and MCP with
consistent behavior.

### List

Lists hosted images you own. Supports optional filters by `source`
(`uploaded` or `generated`) and `visibility`, plus cursor pagination.

- CLI: `goodeye images list` (add `--source`, `--visibility`, `--json`)
- MCP tool: `list_images`
- REST: `GET /v1/images`

### Get

Returns one image record in full: URL, visibility, source, expiry, and
timestamps. Owner-only; a record you do not own returns 404 (existence
masking).

- CLI: `goodeye images get <image_id>` (add `--json`)
- MCP tool: `get_image`
- REST: `GET /v1/images/{image_id}`

### Update

Updates visibility and/or expiry on an image you own. All fields are
optional; only the ones you pass change.

- CLI: `goodeye images update <image_id>` (add `--visibility`,
  `--ttl-seconds`, `--permanent`, `--clear-ttl`, `--json`)
- MCP tool: `update_image`
- REST: `PATCH /v1/images/{image_id}`

### Delete

Permanently removes a hosted image. The URL stops resolving immediately and
the underlying storage is reclaimed. There is no recovery path.

- CLI: `goodeye images delete <image_id>` (`--yes` to skip the prompt)
- MCP tool: `delete_image`
- REST: `DELETE /v1/images/{image_id}`

## See also

- [Image generators](image-generators.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
