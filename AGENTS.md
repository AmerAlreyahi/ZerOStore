# ZerOStore: notes for agents and maintainers

ZerOStore is the app store of [ZerOS](https://github.com/AmerZuher/ZerOS). Every ZerOS server
downloads one file from here, `catalog.json`, and shows the pictures this repository publishes
through GitHub Pages (`https://amerzuher.github.io/ZerOStore/`). Nothing else is read at run time.

`catalog.json` is **built, never edited by hand**. Change the inputs below, rebuild, check, push.

## Files

| File | What it is | Edit by hand? |
| --- | --- | --- |
| `catalog.json` | The catalog servers download: every app's recipe, rating, pictures, the front page. | No, built |
| `zerostore.json` | Aliases between the two upstream stores, and compatibility findings by app ID. | Yes (see below) |
| `storefront.json` | The Store's front page: spotlight apps, sections, each category's featured apps. | Yes |
| `categories.json` | One sentence per category, by category name. | Yes |
| `text.json` | Hand-written fixes per app ID: `title`, `tagline`, `description`, `setupNotes`, `developer`, `icon`. | Yes |
| `icons/`, `screenshots/`, `heroes/` | Pictures the build fetched and resized. | No, built |
| `.media-misses.json` | Apps the build found no picture for, remembered 30 days. | No |
| `.nojekyll` | Tells GitHub Pages to serve files as they are. | Keep it |
| `NOTICE`, `LICENSE`, `LICENSES/` | Where the content comes from, and its licences. | When sources change |

## Updating the store

The tool lives in the ZerOS repository (`backend/cmd/zeros-storecheck`). Go 1.26 or later.

```sh
cd <ZerOS checkout>/backend
go build -o /tmp/storecheck ./cmd/zeros-storecheck
/tmp/storecheck build -store <ZerOStore checkout>
```

The build downloads the CasaOS and Umbrel stores, merges them (an Umbrel app wins a twin), names
every app, rewrites text that names another platform, finds pictures, runs every app through the
ZerOS agent's Compose review, applies `zerostore.json`, and writes `catalog.json` only if a server
would read every app back. It takes about two minutes when the pictures are already there.

Useful flags:

- `-no-media`: fetch no pictures; use what the checkout has (fast; for text or storefront changes).
- `-refresh-media`: fetch every picture again (slow; only when pictures are wrong or stale).

Read the last lines of its output. A good build says `0 texts still name another store`. It also
lists apps without an icon, screenshots or a hero (`noicon`, `noscreenshots`, `nohero` lines).

Then check and publish:

```sh
git diff --stat                 # what changed
git add -A && git commit -m "…" && git push
```

GitHub Pages republishes the pictures within a couple of minutes. Servers pick up the catalog at
their next store sync (Settings or the Store's Refresh).

## Common tasks

**Test apps on a real server and record the result.** Use a test ZerOS server, never someone's own:

```sh
/tmp/storecheck install -data zerostore.json -server http://<test server> -user <admin> \
  -password-file <file> -write <app id> <app id> …
```

Each app is installed, kept running 15 seconds, opened through the gateway, and removed with its
data. Results go into `zerostore.json` as `verified` or `broken` for that app's version. Rebuild
afterwards so the catalog carries them.

**Mark an app by hand.** In `zerostore.json` → `apps`, add
`"<id>": {"status": "incompatible", "notes": ["One plain sentence why."]}` with no `source`: the
tools never overwrite a finding without one. Statuses: `verified`, `limited`, `broken`,
`incompatible`. Notes are shown to users: never name another store or platform in them.

**Two stores' apps are the same app but not matched** (or matched wrongly). In `zerostore.json` →
`aliases`, map the Umbrel ID to the CasaOS ID (`"umbrel.trilium-notes": "org.icewhale.trilium"`),
or to `""` to say they are different apps.

**Fix an app's text or give it an icon.** In `text.json`:
`"<id>": {"tagline": "…", "icon": "https://…/logo.svg"}`. The icon is fetched into `icons/`.

**Change the front page.** Edit `storefront.json`, then build with `-no-media`:

- `spotlight`: app IDs for the top carousel; only apps with a hero picture appear.
- `sections`, in order. `type` is `appList` (with `layout` `rail` or `grid`, `title`, `subtitle`,
  optional `description`) or `categoryFeature` (with `category` as named in the Store, `title`,
  `description`, `textSide` `left` or `right`; its picture is its first app's hero). `apps` are
  app IDs; `"@verified"` stands for every verified app. The first `appList` and the spotlight
  mark their apps as featured.
- `categories`: category name → app IDs to show first.

Unlisted or unknown IDs are dropped by the build and again by servers, so a typo costs nothing
but an empty spot.

## Rules

- **Never hand-edit `catalog.json` or the picture folders.** The next build overwrites them.
- **App IDs are forever.** Servers remember the ID an app was installed from. A listed app's ID is
  its Umbrel ID without `umbrel.`, or the last part of its CasaOS ID, lowercased. The build keeps
  every app's upstream ID as a former ID, so older IDs keep working. Do not rename apps by hand.
- **Users see ZerOStore only.** No text, note or link may name Umbrel, umbrelOS, CasaOS, IceWhale
  or Zima. The build rewrites what it can (tested in `text_test.go` in ZerOS); fix the rest in
  `text.json`. Real default credentials an app uses (a username `umbrel`) stay as they are.
- **No other store's own artwork.** Pictures come from the CasaOS store (Apache-2.0), from
  dashboard-icons (Apache-2.0), or from each app's own project (its README, its website's share
  picture, its repository's social preview). Do not copy Umbrel's gallery
  (`getumbrel/umbrel-apps-gallery`), its storefront (apps.umbrel.com) or umbrelOS's interface
  pictures. The maintainer decided this on 2026-09-27. Any new source goes into `NOTICE`.
- **Keep the catalog readable by servers already out there.** Servers read `schema` 1: only add
  fields, never rename or remove one, and never change what an existing field means. A breaking
  change needs a new schema number and a ZerOS release that reads it first.
- **Keep the repository under 1 GB**, GitHub Pages' limit (it is about 85 MB today). Pictures are
  resized by the build; do not add originals.

## Where things are explained

- ZerOS `docs/backend-contract.md` → "Implemented (backend): ZerOStore": the catalog format, what
  servers check, IDs and former IDs, the front page, and the test results so far.
- ZerOS `backend/cmd/zeros-storecheck/`: the build (`build.go`), text rules (`text.go`), pictures
  (`media.go`, `discover.go`), front page and categories (`storefront.go`).
- ZerOS `api/openapi.yaml` → the `store` tag: what the Store's frontend receives.
