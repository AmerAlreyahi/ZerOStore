# ZerOStore

ZerOStore is the app store of [ZerOS](https://github.com/AmerZuher/ZerOS). It combines two
existing stores, the [CasaOS App Store](https://github.com/IceWhaleTech/CasaOS-AppStore) and
[Umbrel's app store](https://github.com/getumbrel/umbrel-apps), and adds what ZerOS knows about
whether each app works.

This repository holds **only ZerOS's own data**. It contains no app recipes from either store:

| File | What it is |
| --- | --- |
| `zerostore.json` | Aliases between the two stores and compatibility findings for their apps. |
| `review.tsv` | The latest static review: apps that ZerOS's Compose review refuses, one per line. |

## How a server builds the store

Every ZerOS server downloads both upstream stores itself when it syncs:

1. It reads the CasaOS store's release archive, whose recipes are ordinary Compose files.
2. It reads Umbrel's repository and converts each app into the same form. The conversion removes
   umbrelOS's proxy service and publishes the app's own web port, gives each container the
   `<app>_<service>_1` name its settings expect, writes in the plain constants from `exports.sh`,
   and leaves the umbrelOS variables (`APP_DATA_DIR`, `APP_SEED`, `APP_PASSWORD`,
   `DEVICE_DOMAIN_NAME`, `DEVICE_HOSTNAME`, `APP_DOMAIN`) for the install to fill in. The seed and
   password come from the server's own key, so they are never stored.
3. It rates each Umbrel app while converting it:
   - **incompatible**: it can't run outside umbrelOS. It needs other Umbrel apps or Tor, it was
     withdrawn, it reads a settings file that umbrelOS's setup script writes, it builds from
     source, or it uses umbrelOS settings ZerOS can't provide.
   - **limited**: it installs, but something doesn't carry over (it needs HTTPS, or umbrelOS runs a
     setup script for it).
   - **untested**: nothing known stops it.
4. It merges the two stores. When the same app is in both, the Umbrel version is listed and the
   CasaOS one is hidden, unless the Umbrel version can't work on ZerOS. Two apps count as the same
   when the last part of their IDs matches (`org.icewhale.adguardhome` and `adguard-home`), or when
   `aliases` in `zerostore.json` says so.
5. It applies `zerostore.json`'s findings. Apps that are **broken** or **incompatible** are not
   listed, but they can still be looked up by ID, so an installed app keeps its recipe.

A copy of `zerostore.json` is built into every ZerOS release. A server needs nothing from this
repository at run time.

## `zerostore.json`

```json
{
  "schema": 1,
  "updatedAt": "2026-09-26T00:00:00Z",
  "aliases": { "umbrel.trilium-notes": "org.icewhale.trilium", "umbrel.node-red": "" },
  "apps": {
    "umbrel.uptime-kuma": {
      "status": "verified",
      "version": "2.0.2",
      "testedAt": "2026-09-26T00:00:00Z",
      "notes": ["Installed, kept running and its web interface answered 302."],
      "source": "install"
    }
  }
}
```

- `aliases` maps an Umbrel store ID to the CasaOS store ID of the same app, where the names don't
  match. An empty value marks two apps whose names match but which are different apps.
- `apps` holds findings by store ID. `status` is one of `verified`, `limited`, `broken` or
  `incompatible`. A `verified` or `limited` finding applies only to the `version` it was tested
  at. A `broken` or `incompatible` finding stands until someone tests the app again.
- `source` says what produced the finding: `review` (the static check, replaced on every run),
  `install` (a live install), or nothing for a maintainer's own finding, which the tools never
  overwrite.

## Checking apps

The checks run with `zeros-storecheck`, which is built from the ZerOS repository
(`go build ./cmd/zeros-storecheck` in `backend/`):

```sh
# Static review: runs every app through the Compose review the ZerOS host agent runs
# before an install, and records the apps it refuses.
zeros-storecheck review -data zerostore.json -write > review.tsv

# Live check: installs apps on a real ZerOS server through its API, waits for them to keep
# running and answer on their web port, removes them, and records the result.
zeros-storecheck install -data zerostore.json -server http://<server> -user <admin> \
  -password-file <file> -write umbrel.uptime-kuma org.icewhale.glances
```

Use a test server for the live check. Each app is installed with its data and then removed again.

To ship updated data, copy `zerostore.json` into the ZerOS repository
(`make zerostore ZEROSTORE=<path to this repository>` in `backend/`) and release ZerOS.

## Licences

ZerOStore's own data is available under the MIT licence (see `LICENSE`).

The upstream stores are not part of this repository and are not redistributed by it:

- The CasaOS App Store is published by IceWhale Technology under the Apache License 2.0.
- Umbrel's app store is published by Umbrel without a licence. Each ZerOS server downloads it from
  Umbrel's repository for its own use, in the same way it downloads the CasaOS store.

App names and trademarks belong to their owners. See `NOTICE`.
