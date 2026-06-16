# Share-Router-App

Native Kotlin/Compose-App (`ilorenz00/share-router-android`). Man teilt beliebigen
Inhalt (Bild/Video/Audio/Datei/URL/Text) an *Share Router*; die App liest die
OpenAPI-Spec der MasterAPI, bestimmt die logische Art des Inhalts und bietet jeden
passenden Endpoint an. Auslieferung über dieses F-Droid-Repo
([release-pipeline.md](release-pipeline.md)).

## Datenfluss (Senden)

```
ShareActivity → IntentParser (klassifiziert URI/Text → InputKind)
  → ShareViewModel.dispatch → OpenApiFetcher (Spec) + EndpointMatcher
  → Dispatcher.send (baut multipart/json/… aus x-share-Annotationen)
  → Antwort: direkt ODER Ticket (async)
```

Matching wird von **x-share-Vendor-Extensions** in der Spec gesteuert
(`x-share-accepts`, `x-share-field`, `x-share-style`, `x-share-url-hosts`,
`x-share-defaults`), mit Schema-Inferenz als Fallback. Auth: OIDC/Authentik
(siehe [cluster-and-api.md](cluster-and-api.md)).

## Features

**Settings-Overlay** (intuitiv sortiert): oben **Login Authentik** + eingeklapptes
Dropdown „config Authentik / OIDC"; darunter **„Tailscale aktiv?" + Test** +
eingeklapptes „config MasterAPI". Toolbar-Icon öffnet den Verlauf.

**Antwort-Notifications + Verlauf:** Jede API-Antwort wird erfasst und
- inline im Share-Sheet mit **Copy / Save** gezeigt,
- als **Notification** (Kanal *API responses*) mit **Copy** (Zwischenablage) und
  **Save** (→ `Downloads/`) gepostet; Tippen öffnet den Verlauf.
Der volle Body wird **persistent** gespeichert (JSON in `filesDir`, neueste zuerst,
max. 200) und ist im Verlauf abrufbar — auch nach dem Wischen der Notification, mit
denselben Aktionen + Löschen. Implementierung: `data/history/HistoryStore.kt`,
`notify/Notifications.kt` + `NotificationActionReceiver.kt`,
`data/export/ResponseExport.kt`, `ui/HistoryScreen.kt`.
Berechtigung: `POST_NOTIFICATIONS` (Laufzeit-Anfrage ab Android 13).

**Async GPU-Tickets:** Antwortet ein Endpoint mit `{"ticket","status":"queued",…}`,
behandelt die App das **nicht** als Ergebnis, sondern pollt im Hintergrund
(**WorkManager**, `work/JobTicketWorker.kt`, `GET /api/v2/gpu/jobs/{ticket}`) bis
`done/failed/cancelled` — überlebt das Schließen des Share-Sheets und Prozess-Tod,
setzt nach WorkManagers ~10-Min-Cap per `Result.retry()` fort. Bei `done` wird das
Ergebnis **inhaltsoffen** gerendert: dekodierbares Bild (SD `images[]`/base64/
`data:`-URI) → Bild in Notification (BigPicture) + Verlauf; sonst Text/JSON.
Queue-agnostisch (sd/ephemeral/künftige kinds) — neuer Medientyp nur über
`extractImageBytes()` in `data/export/ResponseExport.kt`.

## Bild-Upload (robust)

Die MasterAPI leitet Uploads via Go `CreateFormFile` als `application/octet-stream`
weiter → der Downstream erkennt das Bild am **Dateinamen-Suffix**. Daher in
`Dispatcher`:
- Dateiname bekommt immer eine echte Endung (aus dem MIME-Typ, falls `DISPLAY_NAME`
  keine hat — Google Photos etc.),
- MIME konkret über `ContentResolver` (kein Wildcard `image/*`, das `toMediaType()`
  crashen ließe),
- `readBytes`: Fallback auf `openAssetFileDescriptor`, und **kein stiller Leer-
  Upload** mehr (leere/unlesbare Datei → klare Fehlermeldung),
- Diagnose-Anzeige „↑ hochgeladen: X" im Share-Sheet.

Hinweis: Ein „kein Bild"-Fehler trotz korrektem Upload war serverseitig (Vision-
Kontextlimit) — Story in [cluster-and-api.md](cluster-and-api.md).

## Tests / Build

`gradle assembleDebug` bzw. `assembleRelease` (Letzteres = CI-Lint). AVD
`shareRouterTest` mit `-dns-server 192.168.11.152` starten (sonst kein internes
DNS). `mock/` enthält eine Stand-in-MasterAPI für Emulatortests. Details:
[toolchain.md](toolchain.md).
