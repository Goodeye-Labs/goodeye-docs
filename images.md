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

**MCP tool.** `upload_image` accepts the image bytes as a base64-encoded
string in the `data` parameter, alongside `visibility` and `ttl_seconds`.

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
- **Private image**: requires the owner's credentials, or a valid view link. A
  view link appends the image's capability secret as a `?k=` query parameter
  (`/v1/i/{token}.{ext}?k=<secret>`), so it opens in any browser and can be
  forwarded to people you choose, while the bare URL stays locked. An anonymous
  request with neither returns `401 auth_required`; a request authenticated as
  someone other than the owner returns `404 image_not_found` (the same as a
  missing image, so ownership is not revealed).
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

### Viewing and sharing a private image

A private image still has a browser-viewable **view link**: a URL that carries
the image's capability secret as a `?k=` query parameter. Anyone you hand that
link to can open the image, while the bare URL (without the secret) stays
locked, so you do not have to make an image public to let a few people see it.

- `get_image` (CLI `goodeye images get <image_id>`, REST
  `GET /v1/images/{image_id}`) returns the view link as the `url` field for a
  private image you own. Listing images never includes the secret; fetch a
  single image to get its view link.
- To revoke links you shared earlier, rotate the secret: CLI
  `goodeye images reset-link <image_id>` (or `goodeye images update <image_id>
  --rotate-view-secret`), MCP `update_image` with `rotate_view_secret: true`, or
  REST `PATCH /v1/images/{image_id}` with `{"rotate_view_secret": true}`. Every
  previously shared link stops working and a fresh view link is issued.

### Content screening on public images

A public URL is readable by anyone, so Goodeye screens an image for
disallowed content whenever you make it public: on a public upload and on a
flip from private to public. Images that stay private are never screened.

- If the screen finds disallowed content, the request is rejected with
  `422 image_content_rejected` and the image is not made public.
- If the screen cannot run because it is temporarily unavailable, the request
  fails with `503 image_screening_unavailable` rather than publishing content
  that was not screened. Retry shortly.

A private image and private-only changes (setting a TTL, making an image
private) are never affected.

## TTL and permanence

An image can have an expiry time (a TTL), be explicitly permanent, or simply
have no TTL set (it stays until you delete it).

- **Set a TTL**: the image stops resolving after the given number of seconds
  from the moment the TTL is applied (not from upload time).
- **Clear a TTL**: the image stays until you delete it or set a new TTL.
- **Mark permanent**: clears any expiry so the image stays until you delete
  it. This is a convenient way to remove a TTL outright; the result is the
  same as an image that has no TTL set.
- `ttl_seconds` and `permanent` are mutually exclusive on any single call.

```sh
# Extend to 30 days from now
goodeye images update <image_id> --ttl 2592000

# Remove the expiry and keep the image indefinitely
goodeye images update <image_id> --permanent
```

When an image expires, its URL stops resolving (404). The record is not
immediately purged; the storage is reclaimed automatically later.

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

Authenticated generations are hosted as **public** images by default with no
expiry, so the `url` opens in any browser and persists until you delete it. Pass
`visibility=private` (CLI `--visibility private`) to keep an image private
instead; its `url` is then a view link only you can open and forward (see
"Viewing and sharing a private image" above). Anonymous generations are always
stored as **public** images with a short TTL; the URL is accessible without
credentials but expires automatically.

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

Updates visibility, expiry, or the private view link on an image you own. All
fields are optional; only the ones you pass change. Pass `--rotate-view-secret`
(MCP/REST `rotate_view_secret: true`) to issue a fresh view link and revoke the
links you shared earlier.

- CLI: `goodeye images update <image_id>` (add `--visibility`,
  `--ttl`, `--permanent`, `--rotate-view-secret`, `--json`)
- MCP tool: `update_image`
- REST: `PATCH /v1/images/{image_id}`

### Delete

Permanently removes a hosted image. The URL stops resolving immediately and
the underlying storage is reclaimed. There is no recovery path.

- CLI: `goodeye images delete <image_id>` (`--yes` to skip the prompt)
- MCP tool: `delete_image`
- REST: `DELETE /v1/images/{image_id}`

## Shortcut commands

Four convenience commands wrap the most common `update` operations so you
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

### reset-link

Issue a fresh view link for a private image and revoke the links you shared
earlier, in one step.

```sh
goodeye images reset-link <image_id>
```

Equivalent to `goodeye images update <image_id> --rotate-view-secret`. Prints
the new view link. Accepts `--json` to print the updated record.

## Error codes

Uploading and managing images can return these errors. Most carry a stable
`error` slug and a human-readable `message`; the one exception is noted in the
table.

| HTTP status | Slug | When it occurs |
|-------------|------|----------------|
| 401 | `auth_required` | No credentials on an upload, management, or private-image request |
| 404 | `image_not_found` | The image does not exist, has expired, or is owned by someone else (existence is masked) |
| 413 | `file_too_large` | The uploaded file exceeds the per-file size limit |
| 415 | `unsupported_image_type` | The bytes are not a PNG, JPEG, WebP, or GIF |
| 422 | `quota_exceeded` | Storing the image would exceed your storage quota |
| 422 | `image_content_rejected` | A public upload or a flip to public was screened and found to contain disallowed content; the image is not made public |
| 422 | (no slug) | An update supplied both `ttl_seconds` and `permanent`, which are mutually exclusive. This case returns a plain `detail` message rather than the `{error, message}` envelope. |
| 503 | `image_screening_unavailable` | A public upload or flip could not be screened because the screen was temporarily unavailable; the image was not made public. Retry shortly |

## See also

- [Image generators](image-generators.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
