# Release-Pipeline (vollautomatisch)

Eine Update-Auslieferung ist nur ein **Push auf `main`** des App-Repos. Alles
Weitere läuft ohne manuellen Schritt bis aufs Handy.

## Ablauf

```
git push  (App-Repo, main)
  │
  ├─ CI "release-apk" (.github/workflows/fdroid.yml)
  │    • baut signiertes Release-APK
  │      versionCode = github.run_number, versionName = 0.1.<run_number>
  │    • gh release create  → GitHub Release
  │    • repository_dispatch (event "app-updated") an fdroid-lorenzl5   ← DISPATCH_TOKEN
  │
  ├─ CI "build-fdroid-repo" (dieses Repo, .github/workflows/build-repo.yml)
  │    • lädt das neueste Release-APK jeder App in APPS
  │    • fdroid update  (signiert den Index mit dem Repo-Keystore)
  │    • publish nach gh-pages
  │
  └─ Phone: F-Droid (abonniert https://ilorenz00.github.io/fdroid-lorenzl5/repo)
       sieht den höheren versionCode → Update
```

Ohne `DISPATCH_TOKEN` würde der zentrale Build nur per 30-Minuten-Schedule laufen
(statt sofort). Mit Token ist es event-driven und ohne Leerlauf.

## DISPATCH_TOKEN (der Auto-Trigger)

Der letzte Schritt der App-CI feuert ein `repository_dispatch` an dieses Repo —
authentifiziert über das Secret **`DISPATCH_TOKEN`** im App-Repo.

**Token-Anforderung** (die Dispatch-API lehnt schwächere mit `403 contents=write` ab):

- **Fine-grained PAT** (empfohlen): Repository-Zugriff = `fdroid-lorenzl5`,
  Permission **Contents: Read and write**. **Ein Token gilt für alle Apps.**
- oder **classic PAT** mit `repo`-Scope.

Setzen / rotieren:

```sh
gh secret set DISPATCH_TOKEN -R ilorenz00/<app-repo>   # PAT einfügen
# alten PAT danach in den GitHub-Token-Settings widerrufen
```

Fine-grained PATs lassen sich in den Settings **bearbeiten** (Permission ändern)
ohne dass sich der Token-Wert ändert — ein bereits gesetztes Secret bleibt gültig.

## Neue App anbinden — 3 Schritte

1. **Release-Workflow:** `.github/workflows/fdroid.yml` aus `share-router-android`
   1:1 kopieren (baut signiertes APK, `gh release create`, `repository_dispatch`).
2. **Secrets im neuen App-Repo** (überall **dieselben Werte**):
   ```sh
   gh secret set DISPATCH_TOKEN       -R ilorenz00/<app>   # derselbe PAT
   gh secret set SIGNING_KEYSTORE_B64 -R ilorenz00/<app>
   gh secret set SIGNING_STORE_PASSWORD -R ilorenz00/<app>
   gh secret set SIGNING_KEY_ALIAS    -R ilorenz00/<app>
   gh secret set SIGNING_KEY_PASSWORD -R ilorenz00/<app>
   ```
3. **Hier registrieren:** eine `owner/repo`-Zeile zu `APPS` in
   `.github/workflows/build-repo.yml` hinzufügen, committen.

## Signing

APKs **und** der Repo-Index werden mit demselben Keystore signiert
(`D:\android\sharerouter-release.jks`, Alias `sharerouter`, Credentials im
Passwortmanager). Secrets `SIGNING_KEYSTORE_B64`, `SIGNING_STORE_PASSWORD`,
`SIGNING_KEY_ALIAS`, `SIGNING_KEY_PASSWORD` liegen hier (Index-Signatur) **und**
in jedem App-Repo (APK-Signatur). **Keystore-Verlust = F-Droid lehnt alle künftigen
Updates ab (Signaturwechsel) → unbedingt sichern.**

## Troubleshooting

- **„Kein Update verfügbar" trotz Push:** Index prüfen —
  `curl -s https://ilorenz00.github.io/fdroid-lorenzl5/repo/index.xml | grep -o '<versioncode>[0-9]*'`.
  Bleibt der versionCode alt, lief der zentrale Build nicht. Manuell anstoßen:
  `gh workflow run build-fdroid-repo -R ilorenz00/fdroid-lorenzl5`.
- **Dispatch-Schritt grün, aber kein zentraler Build:** `DISPATCH_TOKEN` fehlt
  oder hat zu wenig Rechte (siehe oben, `403 contents=write`).
- **GitHub Pages hängt nach:** nach dem zentralen Build folgt noch eine
  „pages build and deployment" (~30–60 s), bis der Index öffentlich neu ausliefert.
- **Lokal wie CI bauen:** `gradle assembleRelease` (führt `lintVitalRelease` aus —
  dieselbe Prüfung wie die CI). Siehe [toolchain.md](toolchain.md).
