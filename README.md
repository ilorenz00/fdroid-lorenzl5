# fdroid-lorenzl5

Central F-Droid repository for all lorenzl5 Android apps. Subscribe **once**
on the phone (F-Droid → Settings → Repositories → `+`):

```
https://ilorenz00.github.io/fdroid-lorenzl5/repo
```

Every app added here shows up automatically — no new source needed per app.

📖 **Volle Doku:** [`docs/`](docs/README.md) — Release-Pipeline, Cluster/API,
die Share-Router-App und die lokale Toolchain.

## How it works

- Each app repo builds a **signed release APK** in its own CI and publishes it
  as a GitHub Release (see `share-router-android/.github/workflows/fdroid.yml`).
- This repo's workflow (`build-repo.yml`) runs every 30 minutes (plus manual
  `workflow_dispatch` and instant `repository_dispatch`), downloads the latest
  release APK of every repo in its `APPS` list, runs `fdroid update` over the
  full set and publishes the index to GitHub Pages (`gh-pages` branch).

## Adding a new app — full automation in 3 steps

Every new app gets the same hands-off pipeline as `share-router-android`:
**push → signed release → instant index rebuild → phone update**, no manual step.
For a new app repo:

1. **Release workflow:** copy `.github/workflows/fdroid.yml` from
   `share-router-android` verbatim. It builds the signed APK, runs
   `gh release create`, and (last step) fires a `repository_dispatch` to this
   repo — the instant trigger is already baked in.
2. **Secrets in the new app repo** — reuse the **same values** across all apps:
   ```sh
   gh secret set DISPATCH_TOKEN       -R ilorenz00/<new-app-repo>   # same PAT everywhere
   gh secret set SIGNING_KEYSTORE_B64 -R ilorenz00/<new-app-repo>
   gh secret set SIGNING_STORE_PASSWORD -R ilorenz00/<new-app-repo>
   gh secret set SIGNING_KEY_ALIAS    -R ilorenz00/<new-app-repo>
   gh secret set SIGNING_KEY_PASSWORD -R ilorenz00/<new-app-repo>
   ```
   The `SIGNING_*` set signs the APK; `DISPATCH_TOKEN` is the **one** PAT that
   triggers the instant rebuild (see *Instant rebuilds* — the same token value
   works for every app). Without `DISPATCH_TOKEN` the app still lands on the
   30-min schedule, just not instantly.
3. **Register it here:** add one `owner/repo` line to `APPS` in
   `.github/workflows/build-repo.yml`, commit. Done — that app's next release
   rebuilds and republishes the index automatically.

## Signing

APKs and the repo index are signed with the shared keystore
(`D:\android\sharerouter-release.jks`, credentials in the password manager).
Secrets `SIGNING_KEYSTORE_B64`, `SIGNING_STORE_PASSWORD`, `SIGNING_KEY_ALIAS`,
`SIGNING_KEY_PASSWORD` are set both here (index signing) and in each app repo
(APK signing). **Losing the keystore breaks updates for all installed apps.**

## Instant rebuilds (the DISPATCH_TOKEN) — standard

Each app repo's release workflow triggers an immediate index rebuild here by
sending a `repository_dispatch` (event `app-updated`) authenticated with the
`DISPATCH_TOKEN` secret. This is the **standard** setup (configured for
`share-router-android`); set the same secret on every app repo.

**Token requirements** — the dispatch API rejects anything weaker with
`403 contents=write`:

- **Fine-grained PAT** (recommended): Repository access = `fdroid-lorenzl5`,
  Permission **Contents: Read and write**. **One token covers all apps.**
- or a **classic PAT** with the `repo` scope.

Rotate with `gh secret set DISPATCH_TOKEN -R ilorenz00/<app-repo>` on each app
repo (and revoke the old PAT). The workflow step (already in the app template):

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
