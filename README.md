# colmena-casaos-appstore

Custom [CasaOS](https://www.casaos.io/) App Store for [Colmena](https://github.com/coolabnet/colmena-installer),
an open-source platform for community radio stations and podcasters.

This repo is metadata-only: a `category-list.json` plus one app definition
under `Apps/Colmena/`. It has no build step and no CI — it just points
CasaOS at the pre-built `communityfirst/colmena-app` image published from
[colmena-unified](https://github.com/coolabnet/colmena-unified).

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

- `Apps/Colmena/screenshots/` is currently empty - needs real screenshots
  captured from a running instance.
- Icon (`Apps/Colmena/icon.png`) is a first pass generated from the existing
  bee-glyph mark in `colmena-os`/`backend`, not a final reviewed asset.
- v2: publish public images for `nextcloud` and `mail` to restore full
  feature parity with the multi-container deployment.
