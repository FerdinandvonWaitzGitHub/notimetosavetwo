Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

S3-compatible API at `/storage/v3/s3` so third-party S3 SDKs work against Storage (F-STOR-6). TUS endpoint for resumable uploads (F-STOR-7). Image transformations on signed URLs `?width=…&height=…&format=…` (F-STOR-8). CDN passthrough (`Cache-Control` + origin shielding). S3 backend mode (alternative to FS) with `NTTS_STORAGE_BACKEND=s3` env + bucket config.

## Acceptance criteria

- [ ] `aws-sdk` (v3) against `/storage/v3/s3` can list / put / get / delete objects
- [ ] TUS resumable upload completes a 100MB file across 3 segments with one mid-upload pause
- [ ] Image transform `?width=200&format=webp` returns a resized webp from a JPG/PNG source
- [ ] `Cache-Control` headers passed through; CDN passthrough verified
- [ ] S3 backend mode persists to configured external bucket; FS-mode → S3-mode switch documented and tested

## Blocked by

- [27-storage-core.md](./27-storage-core.md)
