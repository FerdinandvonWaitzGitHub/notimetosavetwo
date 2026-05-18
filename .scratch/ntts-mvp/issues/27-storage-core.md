Status: ready-for-agent

## Parent

[PRD-Backend — NoTimeToSave Backend](../../../PRD.md)

## What to build

supabase-storage wrapped at `/storage/v1`. Buckets with constraints, multipart, signed URLs (`POST /object/sign/...?expiresIn=3600`). RLS on objects. FS backend default; `POST /object/:bucket/:path` upload, `GET /object/:bucket/:path` (private, RLS) and `GET /object/public/:bucket/:path` (no auth). Per-bucket constraints + quotas (F-STOR-9): `storage.buckets.file_size_limit` + new `object_count_limit`, enforced at upload. SHA-256 streaming-computed at upload time, persisted to `storage.objects.metadata->>'sha256'` (also in object-list + `GET /object/info/...`); no automatic dedup. `ntts storage buckets quota` CLI surfaces status.

## Acceptance criteria

- [ ] `supabase-js.storage.from('bucket').upload(...)` lands an object (§6 #1 storage)
- [ ] Private `GET /object/...` returns 401/403 if RLS denies
- [ ] Public `GET /object/public/...` works without auth
- [ ] Signed URL via `POST /object/sign/...?expiresIn=3600` is short-lived (3600s)
- [ ] Per-bucket `file_size_limit` + `object_count_limit` enforced at upload
- [ ] SHA-256 of every uploaded object lives in `storage.objects.metadata->>'sha256'`
- [ ] `ntts storage upload/download/buckets` CLI works
- [ ] Hash computation uses the same read-buffer the persister consumes (no extra IO)

## Blocked by

- [09-rest-rls.md](./09-rest-rls.md)
