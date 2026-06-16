# Cluster-Anbindung & API

Die Apps sprechen die **MasterAPI** im k3s-Homecluster an. Der Cluster lebt im
GitOps-Repo `k3s_cluster_l5` (Flux). Diese Seite beschreibt die Erreichbarkeit,
den async GPU-Ticket-Vertrag und den Deploy-Ablauf.

## Erreichbarkeit (DNS, Authentik, OIDC)

- `api.lorenzl5.com` und `auth.lorenzl5.com` sind **interne Split-Horizon-Domains**.
  Sie lösen **nur über den Cluster-DNS `192.168.11.152`** (blocky) auf → beide
  `192.168.11.150` (Traefik LoadBalancer). Öffentliche Resolver (8.8.8.8) kennen
  sie **nicht**.
- Die MasterAPI liegt in Namespace `automation` (Service `master-api:8080`),
  exponiert über eine Traefik-IngressRoute `Host(api.lorenzl5.com)` hinter der
  **Authentik forward-auth**-Middleware. Unauthentifizierte Requests → 302 zur
  Anmeldung.
- Die App authentifiziert per **OIDC (Authorization-Code + PKCE)** gegen Authentik
  (`auth.lorenzl5.com/application/o/share-router/`, Redirect
  `sharerouter:/oauth2redirect`) und schickt das Token, das die outpost durchlässt.
- **Emulator-Falle:** Ein Android-Emulator nutzt nicht automatisch den Cluster-DNS
  → „unable to resolve host". AVD mit `-dns-server 192.168.11.152` starten.

## Async GPU-Jobs (Ticket-Vertrag)

Der Cluster teilt sich wenige GPUs. Manche Endpoints antworten daher **nicht direkt**,
sondern reihen einen Job ein und liefern ein **Ticket**:

```
POST /api/v1/ai/img2img        (oder txt2img / scribble2render)
  → 202  {"ticket":"gpu_…","status":"queued","priority":1}

GET  /api/v2/gpu/jobs/{ticket}  (gleicher Bearer)
  → {"status":"queued|running|done|failed|cancelled", "result": "...", "error": "..."}
```

Bei `done` enthält `result` bei SD-Jobs die rohe SD-JSON
(`{"images":["<base64-PNG>"],…}`). Clients pollen den v2-Endpunkt bis terminal.
Die Share-Router-App macht das generisch im Hintergrund (siehe
[share-router-app.md](share-router-app.md)).

## Deploy-Ablauf (Flux GitOps) — wichtige Falle

`k3s_cluster_l5` reconciled Branch **main**. Mehrere Dienste betten ihren Code als
**ConfigMap** in `deployment.yaml` ein (z. B. `08_ai/image-to-text` → ConfigMap
`image-to-text-code`, key `main.py`, gemountet nach `/code`, gestartet via
`python main.py`).

**Es gibt KEINEN Reloader / keine Checksum-Annotation.** Eine ConfigMap-Änderung
startet den Pod also **nicht** neu — der laufende Prozess behält den alten Code.
Nach Commit+Push einer ConfigMap-/Code-Änderung live schalten:

```sh
flux reconcile kustomization <name> --with-source -n flux-system  # Git ziehen + anwenden
kubectl rollout restart deployment/<name> -n <ns>                 # neuen Code laden
kubectl rollout status  deployment/<name> -n <ns>
```

Vor dem Commit eingebettetes Python validieren: YAML parsen **und** die ConfigMap-
`main.py` per `py_compile` prüfen.

Werkzeuge vorhanden: `flux` v2.8.8, `kubectl` (Docker Desktop). Cluster-Nodes
`192.168.11.20–24`.

## Beispiel: Bildbeschreibung („no image"-Bug, gelöst)

Symptom: Galerie-Bild an `/ai/image-describe` → „kein Bild übergeben". Diagnose:
Die App sendete das Bild korrekt (verifiziert: 3,6 MB), aber `image-to-text`
schickte das **Originalbild in voller Auflösung** an llava → ~16000+ Bild-Tokens
sprengten vLLMs `--max-model-len 8192` (litellm `400 Input length exceeds…`); der
Dienst fiel still auf den Text-LLM zurück und antwortete „kein Bild".

Fix (im Dienst, nicht in der App): vor dem Vision-Call **downscalen**
(`img.thumbnail((768,768))`, JPEG re-encode) → Tokens unter dem Limit. Generelle
Lehre: Bilder vor Vision-LLM-Calls verkleinern; `image-to-text` macht das jetzt in
`describe_scene` (`/api/v1/analyze` nutzt dieselbe Funktion).
