# maestra-io fork of barman

This is a downstream fork of [EnterpriseDB/barman](https://github.com/EnterpriseDB/barman)
maintained for one purpose: shipping `barman-cloud-backup` with the Cloudflare R2
multipart-upload fix from [PR #1161](https://github.com/EnterpriseDB/barman/pull/1161),
which has not been merged or released upstream.

## Why

R2 requires every non-trailing part of a multipart upload to be **the same length**;
upstream barman's `CloudTarUploader.write()` flushes when the buffer *crosses*
`min_chunk_size`, producing parts of `chunk_size + variable overshoot`. Fine on
AWS S3, fatal on R2 (`InvalidPart: All non-trailing parts must have the same length`).

The fix rewrites the flush loop to emit parts of exactly `chunk_size`, holding the
overshoot in the buffer for the next part. See `barman/cloud.py` `_write_data` /
`_upload_part_buffer(is_final=...)`.

## Branches

- `release/3.18.0` — pristine upstream tag, used as the rebase base.
- `main` — `release/3.18.0` + the two fix commits cherry-picked from PR #1161.

When upstream tags 3.19+ with the fix merged, retire this fork.

## Image

Built and pushed by `.github/workflows/release.yml` to:

```
public.ecr.aws/g5f9s8a4/barman:latest
public.ecr.aws/g5f9s8a4/barman:sha-<git-sha>
```

The image is a standalone barman-cloud CLI (Python 3.13 slim, snappy + lz4 + boto3).
To use it as a drop-in replacement for the CNPG plugin sidecar, override
`/venv/lib/python3.13/site-packages/barman/cloud.py` (or rebuild the
plugin-barman-cloud image with this barman wheel installed).
