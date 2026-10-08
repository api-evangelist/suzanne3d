---
name: suzanne3d-generate-mesh-from-photos
description: Reconstruct a 3D mesh from one to four photos with the Suzanne API by staging images through presigned uploads, submitting a photo-to-3D job, polling it, and downloading the result.
api: Suzanne API
provider: suzanne3d
operations:
  - createUpload
  - createPhotoTo3dGeneration
  - getJob
  - downloadModel
contract: openapi/_ae-authored/suzanne3d-openapi-generated.yml
generated: 2026-10-07
method: generated
source: https://console.suzanne3d.com/documentation/photo-to-3d
---

# Generate a mesh from photos

Base URL `https://api.suzanne3d.com`, `Authorization: Bearer <key>` on every call.

1. **Stage each view** — `createUpload` (`POST /v1/uploads`, empty body) returns `upload_id` (`upl_<uuid>`), a presigned `upload_url` valid 5 minutes, and `expires_at`. `PUT` the JPEG/PNG (≤ 20 MB) to `upload_url` **without a Content-Type header** (curl: `-H "Content-Type:"`; requests: `headers={"Content-Type": None}`; axios: `'Content-Type': undefined`), or S3 answers `403 SignatureDoesNotMatch`. Uploaded objects expire after 7 days.
2. **Submit** — `createPhotoTo3dGeneration` (`POST /v1/generations/photo-to-3d`) with `model` and exactly one of `images_upload_ids` (`front` required; `back`/`left`/`right` optional for `sculptor`; `capture` needs all four) or `images_inline` (`front` as base64, single photo ≤ ~5 MB; more than `front` inline is `400 inline_multi_photo_not_supported`). One view routes to single-image reconstruction, 2-4 views to multi-view, automatically. Add `Idempotency-Key: <uuid>`. Response `202` with `job_id`.
3. **Poll** — `getJob` every 5-10 s until terminal; multi-view typically takes 1-4 min. A `failed` job reports `error.code` `missing_image` (an upload id no longer resolves: re-upload) or `insufficient_views` (capture without four valid views).
4. **Download** — `downloadModel` with `format` in the job's `outputs`; follow the `302` to the 15-minute presigned URL and re-request for a fresh one.

Parameters (`faces` menu, `pbr`, `quad` capped at 150000 faces as FBX, `texture_quality`) behave the same as for text jobs. Errors use the `{ "error": { type, code, message, request_id } }` envelope.
