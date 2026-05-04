---
name: vello-assets
description: Use when the user wants to capture, search, retrieve, or share an asset (image, video, document, audio, or other binary file) via Vello. Captures default to the user's personal vault; sharing into a shared space is always an explicit follow-up via share_asset_to_space. Trigger on phrases like "save this image to Vello", "find that infographic I made", "share this PDF to our team space", or any verb-noun pair where the noun is a file and the verb is store/find/share/get/upload.
author: Matteo Stohlman
version: 1.0.0
---

# Vello Assets

Vello stores binary artifacts (images, videos, documents, audio, other) as `assets` rows backed by content-addressed Storage. This skill teaches when and how to use the asset tools so you don't have to re-derive the recipe each session.

## Mental model

- **Personal vault is the default.** An asset captured without `space_id` lands in the user's private vault. *No other user can read it, ever.*
- **Shared spaces are deliberate.** Only set `space_id` if the user explicitly names one ("put this in our Iris design space"). Never infer from context.
- **Sharing is copy-not-move.** When you call `share_asset_to_space`, the original personal-vault row is untouched. The bytes are deduped (content-addressed), so no re-upload happens.
- **Attribution is permanent.** Whoever first captured an asset stays as `owner_user_id` forever, even when you re-share it across spaces.
- **Don't use `bytes_base64`.** It does not exist. There is exactly one upload path (see below).

## The upload recipe (always)

Inline base64 is *not* an option — passing bytes through the conversation burns tokens proportional to file size with no upside. Every asset upload is a four-step ritual:

1. **Read the local file** with the file tool.
2. **Compute its sha256 hex digest** in shell:
   ```sh
   shasum -a 256 <path> | cut -d' ' -f1
   ```
3. **Call `get_signed_upload_url`** with `content_hash`, `mime_type`, and `expected_byte_size`. You get back `upload_url` and `storage_path`.
4. **PUT the bytes** to the upload_url:
   ```sh
   curl -X PUT -H "Content-Type: <mime>" --data-binary @<path> "<upload_url>"
   ```
5. **Call `capture_asset`** with the `storage_path` from step 3, plus `mime_type`, `asset_class`, and any optional metadata.

The `storage_path` always follows the `{first2_of_hash}/{full_hash}.{ext}` convention. If `capture_asset` returns `Storage object not found`, the PUT in step 4 didn't actually succeed — surface the error rather than retrying blindly.

## When to set `source_thought_id`

If the asset was produced by a Panning-for-Gold run, an AI-generated artifact tied to a specific captured idea, or directly traceable to a Vello thought, set `source_thought_id` to that thought's UUID. This builds a backref that's queryable later. Don't synthesize one — only set it when the link is genuine.

## Sharing into a space

Use `share_asset_to_space(source_asset_id, target_space_id)` only when the user explicitly names a target space. Never infer "this seems team-relevant, let me share it." When sharing, warn the user about PII in `title`/`description`/`tags` — those fields are visible to all space members. If the user uses the asset's tags as private notes, surface that risk before sharing.

## Tool reference

- `capture_asset(storage_path, mime_type, asset_class, …)` — register an already-uploaded file's metadata. Land in personal vault unless `space_id` is set. Returns `{asset_id, content_hash, was_dedup}`.
- `get_signed_upload_url(content_hash, mime_type, expected_byte_size)` — pre-issue a Storage upload URL. Returns `{upload_url, storage_path, expires_at}`.
- `search_assets({query, tags, asset_class, scope, space_id, limit})` — find assets by text/tags/class. RLS-scoped to vault + space memberships. *Does not* return signed download URLs — call `get_asset` for that.
- `get_asset(asset_id, signed_url_ttl_seconds?)` — fetch full metadata + a fresh signed download URL. Writes a `view` audit-log entry.
- `list_recent_assets({scope, space_id, asset_class, limit})` — recent assets, metadata only. Same scoping rules as search.
- `share_asset_to_space(source_asset_id, target_space_id)` — copy an asset into a shared space. Original untouched. Attribution preserved.

## Asset classes

Use the most specific class:
- `image` — png, jpg, webp, gif, svg, heic
- `video` — mp4, mov, webm
- `audio` — mp3, wav, ogg, flac, m4a
- `document` — pdf, docx, plain text, markdown, csv
- `other` — anything that doesn't fit the above

## What NOT to do

- Don't pass raw file bytes through tool arguments. There is no `bytes_base64` field.
- Don't combine capture + share in one operation. Capture into the personal vault first; share to a space as a deliberate second step.
- Don't infer that an asset should be shared based on its content. Sharing is always user-initiated.
- Don't put PII into `title`/`description`/`tags` of an asset that will be shared into a space — those fields are visible to all space members.
