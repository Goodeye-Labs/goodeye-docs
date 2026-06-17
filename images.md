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

The accepted formats are PNG, JPEG, WebP, and GIF. The format is determined
by inspecting the image bytes, not the file name or any declared type, so
renaming a file does not change how it is treated. Anything that is not one
of those four raster formats (SVG, AVIF, PDF, and so on) is rejected with a
`415 unsupported_image_type` error.

**CLI.**

```sh
goodeye images upload path/to/image.png \
  --visibility public \
  --ttl 604800
```

Flags:

- `--visibility public|private` (default: `private`): whether the URL is
  publicly readable without credentials.
- `--ttl <n>`: seconds until the image expires and stops resolving.
  Omit to keep the image until you delete it or set a TTL later.
- `--json`: print the full response as JSON.

The command prints the `id` and the stable `url`.

**MCP tool.** `upload_image` accepts the image as a base64-encoded string
alongside `visibility` and `ttl_seconds`.

**REST.**

```http
POST /v1/images
Authorization: Bearer good_live_EXAMPLE_xxxxxxxx
Content-Type: multipart/form-data

file=<binary>
visibility=public
ttl_seconds=604800
```

The response carries `{id, token, url, visibility, expires_at,
size_bytes, content_type}`.

## The serving URL

The `url` in the response is the stable, embeddable link to the raw image
bytes. It follows the pattern:

```
GET /v1/i/{token}.{ext}
```

The `{token}` is the unguessable identifier; the `.{ext}` suffix (for example
`.png`) is cosmetic for embeddability and is ignored by the server. The
content type served always comes from the stored image bytes, never from the
extension you put in the URL.

Behavior on a serving request:

- **Public image**: served to anyone with the URL, including callers with no
  credentials. Responses carry cache headers so browsers and CDNs can hold a
  copy for a bounded window.
- **Private image**: requires credentials for the owner. An anonymous request
  returns `401 auth_required`; a request authenticated as someone other than
  the owner returns `404 image_not_found` (the same as a missing image, so
  ownership is not revealed).
- **Expired image**: once an image's TTL has passed, its URL returns
  `404 image_not_found`, effective immediately on the next request.

Use the management `id` (not the serving `token`) for `get`, `update`, and
`delete`.

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
goodeye images update <image_id> --ttl 2592000

# Remove the expiry and keep the image indefinitely
goodeye images update <image_id> --permanent
```

When an image expires, its URL stops resolving (404). The record is not
immediately purged; cleanup happens asynchronously.

## Durable URLs for generated images

When you call `generate_image` (or `goodeye image-generators generate`), each
output image is automatically hosted and the response includes a `hosted_images`
list aligned 1:1 with the generated outputs (alongside the provider `image_urls`
list, which is unchanged). Each entry in `hosted_images` is either an object or
`null`:

- **Object** `{id, url, visibility, expires_at}`: the image was persisted
  successfully. `url` is the durable hosted link that stays valid after the
  provider session ends. `id` is the manageable image identifier.
- **`null`**: that output could not be auto-persisted (for example, the provider
  URL was not on the fetch allowlist or a transient error occurred); the
  provider URL in `image_urls` is your fallback for that position.

Authenticated generations are stored as **private** images with no expiry, so
they persist until you delete them. Anonymous generations are stored as
**public** images with a short TTL; the URL is accessible without credentials
but expires automatically.

You can update a hosted generated image with `goodeye images update` using the
`id` from the corresponding `hosted_images` entry.

## List, get, update, delete

All five management commands are available from the CLI, REST, and MCP with
consistent behavior.

### List

Lists hosted images you own. Supports optional filters by `source`
(`upload` or `generated`) and `visibility`, plus cursor pagination.

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
  `--ttl`, `--permanent`, `--json`)
- MCP tool: `update_image`
- REST: `PATCH /v1/images/{image_id}`

### Delete

Permanently removes a hosted image. The URL stops resolving immediately and
the underlying storage is reclaimed. There is no recovery path.

- CLI: `goodeye images delete <image_id>` (`--yes` to skip the prompt)
- MCP tool: `delete_image`
- REST: `DELETE /v1/images/{image_id}`

## Shortcut commands

Three convenience commands wrap the most common `update` operations so you
do not have to remember the full flag names.

### share

Make an image public in one step. Anyone with the URL can view it.

```sh
goodeye images share <image_id>
```

Equivalent to `goodeye images update <image_id> --visibility public`.
Accepts `--json` to print the updated record.

### unshare

Restrict an image back to private access (owner only).

```sh
goodeye images unshare <image_id>
```

Equivalent to `goodeye images update <image_id> --visibility private`.
Accepts `--json` to print the updated record.

### set-ttl

Set or clear the expiry on an image without specifying any other fields.

```sh
# Set a new TTL of 7 days
goodeye images set-ttl <image_id> 604800

# Remove the expiry entirely
goodeye images set-ttl <image_id> permanent
```

Pass a positive integer (seconds from now) to set a new expiry, or the
literal word `permanent` to remove the expiry and keep the image
indefinitely. Accepts `--json` to print the updated record.

## Error codes

Uploading and managing images can return these errors. Each carries a stable
`error` slug and a human-readable `message`.

| HTTP status | Slug | When it occurs |
|-------------|------|----------------|
| 401 | `auth_required` | No credentials on an upload, management, or private-image request |
| 404 | `image_not_found` | The image does not exist, has expired, or is owned by someone else (existence is masked) |
| 413 | `file_too_large` | The uploaded file exceeds the per-file size limit |
| 415 | `unsupported_image_type` | The bytes are not a PNG, JPEG, WebP, or GIF |
| 422 | `quota_exceeded` | Storing the image would exceed your storage quota |

## See also

- [Image generators](image-generators.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
