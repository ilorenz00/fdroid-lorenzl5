# fdroid-lorenzl5

Central F-Droid repository for all lorenzl5 Android apps. Subscribe **once**
on the phone (F-Droid → Settings → Repositories → `+`):

```
https://ilorenz00.github.io/fdroid-lorenzl5/repo
```

Every app added here shows up automatically — no new source needed per app.

## How it works

- Each app repo builds a **signed release APK** in its own CI and publishes it
  as a GitHub Release (see `share-router-android/.github/workflows/fdroid.yml`).
- This repo's workflow (`build-repo.yml`) runs every 30 minutes (plus manual
  `workflow_dispatch` and instant `repository_dispatch`), downloads the latest
  release APK of every repo in its `APPS` list, runs `fdroid update` over the
  full set and publishes the index to GitHub Pages (`gh-pages` branch).

## Adding a new app

1. New app repo: copy the release workflow from `share-router-android`
   (build signed APK → `gh release create`). Reuse the SIGNING_* secrets.
2. Add one line to `APPS` in `.github/workflows/build-repo.yml` here.
3. Wait ≤30 min (or trigger manually:
   `gh workflow run build-fdroid-repo -R ilorenz00/fdroid-lorenzl5`).

## Signing

APKs and the repo index are signed with the shared keystore
(`D:\android\sharerouter-release.jks`, credentials in the password manager).
Secrets `SIGNING_KEYSTORE_B64`, `SIGNING_STORE_PASSWORD`, `SIGNING_KEY_ALIAS`,
`SIGNING_KEY_PASSWORD` are set both here (index signing) and in each app repo
(APK signing). **Losing the keystore breaks updates for all installed apps.**

## Instant rebuilds (optional)

App repos can trigger an immediate rebuild instead of waiting for the
schedule by sending a `repository_dispatch` with a PAT (secret
`DISPATCH_TOKEN`, scope `repo` on this repository):

```yaml
- name: Trigger F-Droid repo rebuild
  if: env.DISPATCH_TOKEN != ''
  env:
    DISPATCH_TOKEN: ${{ secrets.DISPATCH_TOKEN }}
  run: |
    curl -sf -X POST \
      -H "Authorization: Bearer $DISPATCH_TOKEN" \
      -H "Accept: application/vnd.github+json" \
      https://api.github.com/repos/ilorenz00/fdroid-lorenzl5/dispatches \
      -d '{"event_type":"app-updated"}'
```
