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

- `release/3.19.1` — pristine upstream **tag**, used as the rebase base.
- `main` — `release/3.19.1` + the two fix commits cherry-picked from PR #1161,
  plus the maestra-only build files (`Dockerfile`, `Dockerfile.sidecar`,
  `.github/workflows/release.yml`, this file).

Retire this fork once [PR #1161](https://github.com/EnterpriseDB/barman/pull/1161)
is merged AND released upstream AND the cnpg sidecar image rebuilds against that
release. As of 2026-08-31 the PR is still open, so 3.19.1 and 3.20.0 do not carry
the fix and the overlay is still required.

## Images

`.github/workflows/release.yml` builds two images on every push to `main` and on
every `v*` tag, and pushes both to public ECR:

| repo | dockerfile | purpose |
|---|---|---|
| `public.ecr.aws/g5f9s8a4/barman` | `Dockerfile` | Standalone `barman-cloud-*` CLI (python:3.13-slim + snappy + lz4 + boto3). Reference / debug aid. |
| `public.ecr.aws/g5f9s8a4/plugin-barman-cloud-sidecar` | `Dockerfile.sidecar` | Drop-in replacement for `ghcr.io/cloudnative-pg/plugin-barman-cloud-sidecar:v0.14.0` (`ARG SIDECAR_BASE`). Same Go `/manager` binary, but `/venv/lib/python3.13/site-packages/barman` replaced with patched 3.19.1. This is the one us-omega's and us-omicron's `sidecarImage` in `maestra-io/fluxcd` `base/cnpg-plugin-barman-cloud` points at. |

Tags emitted (per `docker/metadata-action`):
- `latest` (default branch only)
- `main` (branch ref)
- `sha-<short>` (every commit — pin via this in fluxcd)
- `<semver>` / `<major>.<minor>` (when a `v*` tag is pushed)
