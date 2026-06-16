# Lokale Toolchain (Build & Emulator)

Die komplette Android-Toolchain liegt auf **D:** (C: war fast voll). Das SDK unter
`C:\Users\…\AppData\Local\Android\Sdk` enthält nur `platform-tools` — das echte SDK
ist auf D:.

| Werkzeug | Pfad |
|---|---|
| Android SDK (inkl. emulator, platform-tools) | `D:\android\Sdk` |
| JDK 17 (Temurin) | `D:\android\tools\jdk17` |
| Gradle 8.7 | `D:\android\tools\gradle-8.7` |
| AVD | `shareRouterTest` (Android 14) in `~/.android/avd` |
| adb | `D:\android\Sdk\platform-tools\adb.exe` |
| emulator | `D:\android\Sdk\emulator\emulator.exe` |

## Bauen (PowerShell)

```powershell
$env:JAVA_HOME="D:\android\tools\jdk17"; $env:ANDROID_HOME="D:\android\Sdk"
D:\android\tools\gradle-8.7\bin\gradle.bat assembleDebug    # schneller Kompilier-Check
D:\android\tools\gradle-8.7\bin\gradle.bat assembleRelease  # inkl. lintVitalRelease (= CI)
```

Es gibt **kein `gradlew`** im Repo; immer das Gradle auf D: verwenden. Die CI nutzt
`gradle/actions/setup-gradle@v4` mit Gradle 8.7 — `assembleRelease` lokal spiegelt
den CI-Build-Schritt (nur ohne Signing-Keystore → unsigniertes APK).

## Emulator + App

```powershell
# WICHTIG: -dns-server, sonst löst der Emulator *.lorenzl5.com nicht auf
D:\android\Sdk\emulator\emulator.exe -avd shareRouterTest -dns-server 192.168.11.152
D:\android\Sdk\platform-tools\adb.exe install -r `
  C:\Users\Lorenz\Documents\Github\share-router-android\app\build\outputs\apk\debug\app-debug.apk
```

`mock/` im App-Repo enthält eine Stand-in-MasterAPI (`serve.ps1` serviert
`openapi.json`, loggt Requests inkl. Content-Length nach `requests.log`) — im
Emulator erreichbar unter `http://10.0.2.2:8765/…`.

## Cluster ansteuern

`kubectl` (Docker Desktop) und `flux` v2.8.8 sind installiert. Read-only-Zugriff auf
die MasterAPI ohne Authentik: `kubectl port-forward -n automation svc/master-api
18080:8080`, dann z. B. `curl localhost:18080/swagger/doc.json`
(`/api/v1/*` braucht zusätzlich Auth). Deploy-Ablauf: [cluster-and-api.md](cluster-and-api.md).
