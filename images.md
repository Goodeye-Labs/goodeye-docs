# Image hosting

Goodeye can host images for you and return a stable URL that never changes,
even if you regenerate, re-upload, or the originating session ends. This page
covers uploading images, managing visibility and expiry, and how generated
images get durable hosted URLs automatically.

For the image generation capability itself, see
[Image generators](image-generators.md). For general workflow usage, see
[Workflows](workflows.md).

## Upload and host an image

You supply the image bytes and Goodeye stores them, returning a stable `url`
you can embed or share.

The accepted formats are PNG, JPEG, WebP, and GIF. The format is determined by
inspecting the image bytes, not the file name, so renaming a file does not
change how it is treated. Anything else (SVG, AVIF, PDF, and so on) is rejected
with a `415 unsupported_image_type` error.

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

The response carries the management `id`, the stable `url`, and the image's
visibility and expiry. See [REST API](rest-api.md) for the full field list.

## Visibility and the serving URL

The `url` is a stable, embeddable link to the raw image bytes that keeps serving
the same bytes even after you change the image's metadata. Every image starts
private unless you set `--visibility public` at upload time, and you can flip
visibility at any time without changing the URL:

```sh
goodeye images update <image_id> --visibility public    # share
goodeye images update <image_id> --visibility private   # revoke public access
```

The same update command works via MCP (`update_image`) and REST
(`PATCH /v1/images/{image_id}`). What a request returns depends on the image:

- **Public**: served to anyone with the URL, including callers with no
  credentials.
- **Private**: requires the owner's credentials or a shareable view link (see
  below). An anonymous request with neither returns `401 auth_required`; a
  request authenticated as someone other than the owner returns
  `404 image_not_found`.
- **Expired**: once an image's TTL has passed, its URL stops working
  (`404 image_not_found`).

Use the management `id` (not the serving URL) for `get`, `update`, and `delete`.

### Sharing a private image

A private image still has a **shareable view link**: a link that opens the image
in a browser for anyone you hand it to, while the bare URL stays locked, so you
do not have to make an image public to let a few people see it. `get_image`
(CLI `goodeye images get <image_id>`, REST `GET /v1/images/{image_id}`) returns
the view link as the `url` field for a private image you own; listing images
never includes it. To revoke links you shared earlier, rotate the secret with
`goodeye images reset-link <image_id>` (MCP `update_image` with
`rotate_view_secret: true`, REST `PATCH` with `{"rotate_view_secret": true}`):
every previously shared link stops working and a fresh one is issued.

### Content screening on public images

A public URL is readable by anyone, so Goodeye screens an image for disallowed
content whenever you make it public: on a public upload and on a flip from
private to public. Images that stay private are never screened.

- If the screen finds disallowed content, the request is rejected with
  `422 image_content_rejected` and the image is not made public.
- If the screen cannot run because it is temporarily unavailable, the request
  fails with `503 image_screening_unavailable` rather than publishing content
  that was not screened. Retry shortly.

## TTL and permanence

An image can have an expiry time (a TTL), be explicitly permanent, or simply
have no TTL set (it stays until you delete it).

- **Set a TTL**: the image stops resolving after the given number of seconds
  from the moment the TTL is applied (not from upload time).
- **Clear a TTL**: the image stays until you delete it or set a new TTL.
- **Mark permanent**: clears any expiry so the image stays until you delete it.
- `ttl_seconds` and `permanent` are mutually exclusive on any single call.

```sh
# Extend to 30 days from now
goodeye images update <image_id> --ttl 2592000

# Remove the expiry and keep the image indefinitely
goodeye images update <image_id> --permanent
```

When an image expires, its URL stops working (404).

## Durable URLs for generated images

When you call `generate_image` (or `goodeye image-generators generate`), each
output image is automatically hosted and the response includes a `hosted_images`
list aligned 1:1 with the generated outputs (alongside the provider `image_urls`
list, which is unchanged). Each entry is either an object or `null`:

- **Object** `{id, url, visibility, expires_at}`: the image was persisted
  successfully. `url` is the durable hosted link that stays valid after the
  provider session ends, and `id` is the manageable image identifier.
- **`null`**: that output could not be saved; use the provider URL in
  `image_urls` as a fallback for that position.

Authenticated generations are hosted as **public** images by default with no
expiry, so the `url` opens in any browser and persists until you delete it. Pass
`visibility=private` (CLI `--visibility private`) to keep an image private
instead; its `url` is then a view link only you can open and forward. Anonymous
generations are always stored as **public** images with a short TTL.

You can update a hosted generated image with `goodeye images update` using the
`id` from the corresponding `hosted_images` entry.

## List, get, update, delete

All five management commands are available from the CLI, REST, and MCP with
consistent behavior.

- **List** images you own, with optional filters by `source` (`upload` or
  `generated`) and `visibility`, plus cursor pagination: CLI
  `goodeye images list`, MCP `list_images`, REST `GET /v1/images`.
- **Get** one image record in full (URL, visibility, source, expiry,
  timestamps). Owner-only; a record you do not own returns 404: CLI
  `goodeye images get <image_id>`, MCP `get_image`, REST
  `GET /v1/images/{image_id}`.
- **Update** visibility, expiry, or the private view link on an image you own;
  only the fields you pass change: CLI `goodeye images update <image_id>`
  (`--visibility`, `--ttl`, `--permanent`, `--rotate-view-secret`), MCP
  `update_image`, REST `PATCH /v1/images/{image_id}`.
- **Delete** an image permanently. The URL stops working immediately and there
  is no recovery path: CLI `goodeye images delete <image_id>` (`--yes` to skip
  the prompt), MCP `delete_image`, REST `DELETE /v1/images/{image_id}`.

## Shortcut commands

Convenience wrappers exist for the most common updates: `goodeye images share`,
`goodeye images unshare`, `goodeye images set-ttl`, and
`goodeye images reset-link`. See [CLI](cli.md) for the full list and the
`update` flags each maps to.

## Error codes

Uploading and managing images can return these errors. Each carries a stable
`error` slug and a human-readable `message`.

| HTTP status | Slug | When it occurs |
|-------------|------|----------------|
| 401 | `auth_required` | No credentials on an upload, management, or private-image request |
| 404 | `image_not_found` | The image does not exist, has expired, or is owned by someone else |
| 413 | `file_too_large` | The uploaded file exceeds the per-file size limit |
| 415 | `unsupported_image_type` | The bytes are not a PNG, JPEG, WebP, or GIF |
| 422 | `quota_exceeded` | Storing the image would exceed your storage quota |
| 422 | `image_content_rejected` | A public upload or a flip to public was screened and found to contain disallowed content; the image is not made public |
| 503 | `image_screening_unavailable` | A public upload or flip could not be screened because the screen was temporarily unavailable; the image was not made public. Retry shortly |

## See also

- [Image generators](image-generators.md)
- [Workflows](workflows.md)
- [Templates](templates.md)
- [CLI reference](cli.md)
- [REST API](rest-api.md)
- [Accounts and billing](accounts-and-billing.md)
