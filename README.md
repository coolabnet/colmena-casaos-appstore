# colmena-casaos-appstore

Custom [CasaOS](https://www.casaos.io/) App Store for [Colmena](https://github.com/coolabnet/colmena-installer),
an open-source platform for community radio stations and podcasters.

This repo is metadata-only: a `category-list.json` plus one app definition
under `Apps/Colmena/`. It has no build step and no CI — it just points
CasaOS at the pre-built `communityfirst/colmena-app` image published from
[colmena-unified](https://github.com/coolabnet/colmena-unified).

## Status: verified working end to end

Standalone smoke-testing (no CasaOS, just `docker compose up` with this exact
file) turned up four bugs that stopped `communityfirst/colmena-app` from
booting at all without a live Nextcloud instance - which v1 deliberately
doesn't bundle. All four are now fixed, rebuilt, republished to Docker Hub,
and re-verified against the live `:latest` tag (not just a local build):

1. **nginx never started.** Alpine's stock `nginx.conf` only auto-includes
   `/etc/nginx/http.d/*.conf`, but the Dockerfile dropped the server block
   into `/etc/nginx/conf.d/` (the Debian convention). Fixed in
   `colmena-os/Dockerfile` (`conf.d` -> `http.d`).
2. **`gunicorn` didn't exist in the final image at all.** The final Docker
   stage only copied `site-packages` from the backend-builder stage, never
   `/usr/local/bin` - so the pip-installed `gunicorn` console script never
   made it in. Fixed by adding a `COPY --from=backend-builder
   /usr/local/bin/gunicorn ...` line.
3. **`DEBUG=false` crashed Django at import time** -
   `settings/prod.py` does `bool(int(os.environ.get("DEBUG", 0)))`, which
   only accepts `"0"`/`"1"`. Fixed in this repo's compose (now `DEBUG=0`),
   along with a missing `DATABASE_URL` (required when `STAGE=prod`).
4. **Superadmin bootstrap required a reachable Nextcloud, unconditionally.**
   `create_superadmin` always called out to Nextcloud's API to mint an app
   password; the generated OpenAPI client only caught bad HTTP status
   codes, not connection failures, so any deployment without Nextcloud
   crash-looped forever. Fixed at the source
   (`backend/apps/nextcloud/occ.py` + `create_superadmin.py`): connection
   errors are now caught and the superadmin is created without an app
   password when Nextcloud isn't reachable.

Fixes 1, 2, and 4 live in `colmena-os`'s Dockerfile and `luandro/backend`
(branch `fix/standalone-boot-nextcloud-optional`) - not in this repo. Note
`colmena-os`'s `.gitmodules` now points its backend submodule at
`luandro/backend` instead of the upstream `colmena-project/dev/backend`,
since the fix isn't merged upstream yet and this repo is slated for
retirement anyway (see `colmena-unified` migration plan).

## Add this store to CasaOS

**Via the UI**: Settings -> App Store -> Sources -> Add Source, then enter:

```
https://github.com/coolabnet/colmena-casaos-appstore/archive/refs/heads/main.zip
```

**Via the CLI**:

```bash
casaos-cli app-management register app-store https://github.com/coolabnet/colmena-casaos-appstore/archive/refs/heads/main.zip
```

Then install "Colmena" from the Media category.

## What v1 ships

- `colmena-app` (frontend + backend in one container) + `colmena-postgres`.
- Nextcloud file sync and outbound email are **not** bundled - point
  `NEXTCLOUD_URL` / `EMAIL_HOST` at your own external instances after
  install, or leave them blank.

## Updating this store

1. Edit `Apps/Colmena/docker-compose.yml` (bump `image:` tag if pinning a
   version instead of `latest`, adjust env defaults, etc).
2. If the icon/screenshots change, replace the files under
   `Apps/Colmena/` - they're served straight from this repo via jsDelivr,
   no rebuild step.
3. Commit and push to `main`. CasaOS sources re-fetch the store
   periodically, or the user can force a refresh from Settings -> App Store.

## Known gaps (tracked, not yet done)

- `/DATA/AppData/$AppID/{media,static}` need to be writable by uid 65534
  (`nobody`, the user the backend process runs as) - CasaOS's own AppData
  provisioning should handle this, but if you see `PermissionError` on first
  boot, `chown -R 65534:65534` those two folders.
- `Apps/Colmena/screenshots/` is currently empty - needs real screenshots
  captured from a running instance.
- Icon (`Apps/Colmena/icon.png`) is a first pass generated from the existing
  bee-glyph mark in `colmena-os`/`backend`, not a final reviewed asset.
- v2: publish public images for `nextcloud` and `mail` to restore full
  feature parity with the multi-container deployment.
