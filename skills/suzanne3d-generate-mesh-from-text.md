---
name: suzanne3d-generate-mesh-from-text
description: Turn a text prompt into a downloadable GLB/OBJ/STL/FBX mesh with the Suzanne API by submitting a text-to-3D job, polling it to a terminal state, and downloading the output.
api: Suzanne API
provider: suzanne3d
operations:
  - createTextTo3dGeneration
  - getJob
  - downloadModel
  - cancelJob
contract: openapi/_ae-authored/suzanne3d-openapi-generated.yml
generated: 2026-10-07
method: generated
source: https://console.suzanne3d.com/documentation/quickstart
---

# Generate a mesh from a text prompt

Base URL `https://api.suzanne3d.com`. Every request carries `Authorization: Bearer <key>`; keys start with `sznn_test_` (sandbox) or `sznn_live_` (production) and stay server-side.

1. **Submit** — `createTextTo3dGeneration` (`POST /v1/generations/text-to-3d`) with `model` (`sculptor` for fast game-ready meshes, `atelier` for premium PBR; `capture` is photo-only), a `prompt` of 1-4000 characters, optional `params` (`faces` from the menu 200000 / 500000 / 1000000 / 2000000, `pbr`, `quad`, `texture_quality`) and `outputs` (any subset of glb, obj, stl, fbx; default glb). Send `Idempotency-Key: <uuid>` so a retry returns the original job instead of creating a duplicate (same key, different body is a 409 `idempotency_key_conflict`). The response is `202` with `job_id` and `status: queued`.
2. **Poll** — `getJob` (`GET /v1/jobs/{job_id}`) every 5-10 seconds until `status` is `done`, `failed` or `cancelled`; typical latency is 30 s - 2 min. Give up after 20 minutes. A `409 concurrent_limit_reached` on submit means the account is at its concurrent-jobs ceiling (default 10): back off and retry. Do not retry `validation_error` or other 4xx.
3. **Download** — `downloadModel` (`GET /v1/models/{job_id}/download?format=glb`) answers `302` to a 15-minute presigned S3 URL; follow the redirect. `409 job_not_done` means poll first; `404 not_found` means the format was not in the job's `outputs`.
4. **Undo** — `cancelJob` (`POST /v1/jobs/{job_id}/cancel`) cancels a queued or running job; cancellation is best-effort and a running job stopped in time is refunded. `already_terminal: true` means nothing changed.

Errors are JSON `{ "error": { "type", "code", "message", "request_id" } }`; keep `request_id` for support. A job that ends `failed` carries `error.code` such as `vendor_model_error` (refunded; surface it, do not auto-retry) or `sqs_enqueue_failed` (safe to retry).
