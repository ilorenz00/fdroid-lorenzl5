# lorenzl5 — Dokumentation

Zentrale Doku rund um das F-Droid-Repo, die Apps und ihre Anbindung an den
k3s-Homecluster. Diese Dateien sind das „Gedächtnis" des Projekts — wichtige,
nicht-offensichtliche Abläufe an einem Ort.

| Dokument | Inhalt |
|---|---|
| [release-pipeline.md](release-pipeline.md) | Vollautomatische F-Droid-Auslieferung: Push → signiertes Release → Index-Rebuild → Phone-Update. `DISPATCH_TOKEN`, neue App hinzufügen, Troubleshooting. |
| [cluster-and-api.md](cluster-and-api.md) | Wie die Apps die MasterAPI erreichen (internes DNS, Authentik/OIDC), der async GPU-Ticket-Vertrag, und der Flux-GitOps-Deploy-Ablauf (inkl. ConfigMap-Reload-Falle). |
| [share-router-app.md](share-router-app.md) | Architektur der Share-Router-App, die Features (Settings, Antwort-Notifications + Verlauf, async Tickets, Bild-Upload) und gelöste Bugs. |
| [toolchain.md](toolchain.md) | Lokale Build-/Emulator-Toolchain (auf **D:**), Build-/Lint-Befehle. |

**Kurzfassung der Auslieferung:** Ein `git push` auf `main` einer App reicht — der
Rest passiert von selbst. Details in [release-pipeline.md](release-pipeline.md).
