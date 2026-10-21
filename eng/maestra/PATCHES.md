# Maestra patches

Applied with `git apply`, in file-name order, onto upstream
`gravitational/teleport` **v18.11.1**. Goal: write session recordings straight
to Cloudflare R2 (maestra-io/issues-maestra#1632). Each patch carries its own
unit test (in-memory or local fake S3, no network).

| Patch | Files | Why |
|---|---|---|
| `0001-s3-disable-versioning.patch` | `lib/events/s3sessions/s3handler.go` + tests | R2 answers `ListObjectVersions` with `501 NotImplemented`; upstream calls it on every recording, metadata and thumbnail download and in the upload completer's session-end recovery |
| `0002-events-fixed-size-parts.patch` | `lib/events/stream.go` + test | R2 rejects `CompleteMultipartUpload` with `400 InvalidPart: All non-trailing parts must have the same length`; upstream parts are ≥ 5 MiB of variable size |
| `0003-s3-credentials-dir.patch` | `lib/events/s3sessions/s3handler.go` + test | the AWS SDK reads `AWS_*` env once at start; Vault cloudflare plugin R2 temp creds are non-renewable (~12 h) and VSO rewrites them into a Kubernetes Secret |

## 0001 — `disable_versioning=true`

`downloadOriginalFile` reads the current object (HEAD + ranged GET, no
`VersionId`) instead of picking the oldest version from `ListObjectVersions`.
That covers every caller: `Download`, `DownloadMetadata`, `DownloadThumbnail`
and the completer. The flag is explicit, like `disablesse`, so nothing changes
silently on AWS S3. There is no automatic fallback on `NotImplemented`.

Ceiling: upstream relies on bucket versioning to keep the first copy of a
recording if it gets overwritten. With this flag, an overwrite (a second
`CompleteMultipartUpload` for the same session) replaces the object. Do not
"fix" that with an R2 bucket lock: recordings and session summaries share the
configured prefix, and Teleport must overwrite a pending summary with the final
one, which a lock on the prefix would refuse. Single-object uploads (metadata,
thumbnails) still use `If-None-Match: *`, and R2 honours it (412 on overwrite,
verified live). A duplicate multipart completion for one session is a Teleport
bug, not an attack surface here: the bucket is written only by auth.

## 0002 — fixed-size parts (no parameter)

`slice.shouldUpload` cuts a slice 256 KiB below `MinUploadBytes`. The existing
`slice.reader` padding then brings every non-last part to exactly
`MinUploadBytes` (5 MiB), using the padding the proto stream format already has.
Readers need no changes.

- The 256 KiB reserve bounds what can reach the buffer after the size check:
  one event (≤ 64 KiB, `MaxProtoMessageSizeBytes`) + the unflushed gzip window
  (klauspost BestSpeed, 64 KiB) + one age STREAM chunk when recording encryption
  is on (64 KiB) + block/trailer overhead. If the bound were ever exceeded, the
  part is left unpadded, which is upstream behaviour.
- It applies only when `MinUploadBytes >= 5 MiB`, i.e. the auth server's
  external-storage streamer. The node-local file streamer (128 KiB) and tests
  with small parts are unchanged.
- Unconditional rather than an S3-only option: an option would need plumbing
  from the upload handler through `lib/service` into `ProtoStreamerConfig`.
  Other backends (AWS S3, GCS, Azure) accept padded parts. The cost is
  ≤ 256 KiB of zero padding per 5 MiB part (~5%).

## 0003 — `credentials_dir=<path>`

Sets `Config.CredentialsProvider` to `aws.NewCredentialsCache(dirCredentials(path))`.
On each refresh, `Retrieve` reads these files from `<path>` (the data keys of
Secret `session-records-r2-creds`, mounted as a volume):

| file | required | use |
|---|---|---|
| `access_key_id` | yes | |
| `secret_access_key` | yes | |
| `session_token` | no | omitted for static keys |
| `expiration` | no | RFC3339; credentials expire 5 min before it, otherwise every 30 min |

`Retrieve` resolves the mount's `..data` link once and reads every file from
that directory, so one refresh never mixes two Secret generations. kubelet swaps
that link atomically, and VSO rewrites the Secret at 67% of the TTL, so the
cached creds always refresh before the old ones expire. The URI
parameter takes precedence over External Audit Storage credentials and env vars.

## Resulting `audit_sessions_uri`

```
s3://teleport-maestra-session-records/<prefix>?endpoint=https://<account-id>.r2.cloudflarestorage.com&region=auto&disablesse=true&disable_versioning=true&credentials_dir=/var/run/secrets/session-records-r2
```

| parameter | origin | why |
|---|---|---|
| `endpoint` | upstream | R2 S3 endpoint (the `endpoint` key of the Secret); path-style addressing is the upstream default |
| `region=auto` | upstream | R2 region |
| `disablesse=true` | upstream | R2 answers `PutObject` with `x-amz-server-side-encryption` with 501 (R2 encrypts at rest itself) |
| `disable_versioning=true` | 0001 | no `ListObjectVersions` on R2 |
| `credentials_dir=…` | 0003 | rotating temp creds from the mounted Secret |

## Verified against R2 (21.10.2026, bucket `teleport-maestra-session-records`, prefix `_probe-1632/`, objects deleted afterwards)

- Streamed a 700-event session of random data through `ProtoStreamer` + the
  patched s3 handler, configured from the URI above via `credentials_dir`, with
  `AWS_*` env unset. Parts `[5242880 5242880 5242880 2166955]` (17.1 MiB);
  `CompleteMultipartUpload` OK; `Download` returned the same 17895595 bytes;
  all 700 events decoded byte-identical.
- `UploadMetadata` / `DownloadMetadata` OK. `Download` of a missing session
  returns `NotFound`.
- Upload completer path: R2's `ListMultipartUploads` returns an upload ID that
  differs from the one `CreateMultipartUpload` returned. `ListParts`,
  `UploadPart` and `CompleteMultipartUpload` all work with the listed ID.
- Pending/final summary uploads (`If-None-Match`/`If-Match`) work unpatched.
