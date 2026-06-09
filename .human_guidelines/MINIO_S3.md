# MINIO_S3.md — asset storage via flysystem + MinIO

This project uses **MinIO** in dev/stage and (likely) AWS S3 in production, both reached through Pimcore's flysystem integration. The on-disk asset folder is *not* the source of truth; the configured storage is.

## Composer

```
"league/flysystem-aws-s3-v3": "^3.29"
```

Note: `composer.json` uses a custom GitHub VCS fork of flysystem-aws-s3-v3 (`thephpleague/flysystem-aws-s3-v3`) and excludes the GitLab-mirrored copy — don't be surprised that `composer why` shows that fork.

## Container

Template: `SUPPORT/Docker/Templates/minio.yaml`
Image: `minio/minio:latest`
Ports (internal): `9000` (S3 API) + `9001` (web console)
Data volume: `./STORAGE/minio:/data`
Init script: `SUPPORT/ServerConfigs/minio-entrypoint.sh` (creates buckets / users on container start)

## Pimcore flysystem config

Pimcore 11's flysystem storage abstraction is configured under `config/packages/` (search `pimcore.flysystem` or `flysystem` in `*.yaml`). Each *Pimcore storage key* (`asset`, `thumbnail`, `version`, `recycle_bin`, `admin`, `tmp`) can target a different filesystem.

To inspect what's actually wired:
```bash
grep -r "pimcore_flysystem\|flysystem\|aws-s3\|s3:" /home/andreasmh/Sites/mc-techgroup/PROJECT/pimcore/config/
```

## Project wrapper: `src/S3/StorageWrapper.php`

A thin service that streams assets from any Pimcore Storage:

```php
public function handleFileRequest(Request $request, string $filePath, array $ignores = []): ?Response
```

Behavior:
- Returns `null` when the path has no extension (lets the controller fall through to other routes)
- Skips files matching any regex in `$ignores`
- Detects thumbnails via `image-thumb__<id>__` / `video-thumb__<id>__` patterns
- Streams the file as `StreamedResponse` from the configured Pimcore Storage

Used by frontend controllers that proxy asset URLs through Symfony (so flysystem can resolve cross-storage), e.g. when assets sit in MinIO/S3 but you want Pimcore's permission system in the path.

## Conventions

- **Don't bypass the wrapper.** If you need to read an asset in a controller, go through `Pimcore\Tool\Storage::get('asset')->...` (or `StorageWrapper`), never `file_get_contents` on a hard-coded path.
- **Thumbnail safety:** `Asset::getThumbnail('definitionName')` throws `NotFoundException` for missing definitions — wrap in try/catch in factories so a typo doesn't 500 the page.
- **Local dev URLs:** the MinIO console is at `http://minio:9001` from inside the docker network and a dynamic host port outside (look up via `docker compose port minio 9001`).
- **Bucket creation:** if a deploy adds a new bucket, update `SUPPORT/ServerConfigs/minio-entrypoint.sh` so fresh dev volumes provision it; otherwise dev devs hit `NoSuchBucket` errors.

## Common pitfalls

- Pimcore caches asset URLs aggressively; after switching storage targets, run `any cc` and `any pimcore pimcore:cache:clear --env=prod` (or just `any cc`) so the URL generator picks up new endpoints.
- A wrong AWS region in `.env` causes flysystem to throw `RequestTimeTooSkewed` against MinIO — MinIO ignores region but the AWS SDK signs by it; mismatched signatures look like clock skew.
- The flysystem fork pin matters: `composer update` without the GitHub VCS source falls back to the regular Packagist version, which can subtly break the AWS extension API. Don't remove the `repositories` entry without testing.
