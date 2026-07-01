# TidesUpdater

`TidesUpdater` ist die server-only Variante aus dem Servermodus von `TidesDownloader`.

Developed with love by Dreamleak for TheDarkTides.

Es läuft als Konsolenprozess, startet eine lokale HTTP-API und kann optional automatisch Steam-Manifeste und Depot-Keys aktualisieren.

## Features

- API-Server starten
- API-Key Schutz für alle Endpunkte
- Keys aus `API\Keys` ausliefern
- Manifeste aus `API\manifests\<appId>` als ZIP ausliefern
- automatische Manifest-/Key-Extraktion über hinterlegte Steam-Accounts
- Keys und Manifeste in Github hochladen

## Startparameter

```text
--api-key <key>              API-Key für alle HTTP-Anfragen
--port <number>              Server-Port, Standard aus settings.json oder 8080
--auto-extract               automatische Extraktion aktivieren
--no-auto-extract            automatische Extraktion deaktivieren
--interval-minutes <number>  Intervall für automatische Extraktion
--github-upload              nach erfolgreichem Extract ins GitHub-Repo hochladen
--no-github-upload           GitHub-Upload deaktivieren
--github-owner <owner>       GitHub User oder Organisation
--github-repo <repo>         GitHub Repository-Name
--github-branch <branch>     Ziel-Branch, z. B. main
--github-token <pat>         GitHub Personal Access Token
--help                       Hilfe anzeigen
```

Startparameter überschreiben die Werte aus der Settings-Datei nur für diesen Programmlauf.

Für den GitHub-Token kannst du statt `--github-token` auch eine Environment Variable verwenden:

```powershell
$env:TIDESUPDATER_GITHUB_PAT = "github_pat_dein_token"
```

Alternativ wird auch diese Variable gelesen:

```powershell
$env:GITHUB_TOKEN = "github_pat_dein_token"
```

## Settings-Datei

Die Settings werden direkt aus dem Programmordner geladen.

Falls die Datei im Programmordner noch nicht existiert, kannst du sie manuell anlegen:

```json
{
  "ServerPort": 8080,
  "ApiKey": "dein-api-key",
  "AutomaticExtractionEnabled": true,
  "AutomaticExtractionIntervalMinutes": 30,
  "GitHubUploadEnabled": true,
  "GitHubOwner": "dein-user",
  "GitHubRepo": "dein-repo",
  "GitHubBranch": "main",
  "GitHubToken": ""
}
```

Wichtige Werte:

```text
ServerPort                           HTTP-Port
ApiKey                               geheimer Zugriffsschlüssel für die API
AutomaticExtractionEnabled           true/false für automatische Updates
AutomaticExtractionIntervalMinutes   Update-Intervall in Minuten
GitHubUploadEnabled                  true/false für Upload nach Extract
GitHubOwner                          GitHub User oder Organisation
GitHubRepo                           Repository-Name
GitHubBranch                         Ziel-Branch
GitHubToken                          PAT, besser leer lassen und Env Variable nutzen
```

Hinweis: Das Programm erzwingt mindestens 10 Minuten Intervall für automatische Extraktion.

## API-Ordnerstruktur

`TidesUpdater` sucht den API-Ordner an diesen Stellen:

```text
<Ordner der EXE>\API
<aktueller Arbeitsordner>\API
```

Erwartete Struktur:

```text
API
├── Keys
│   └── <appId>.key
└── manifests
    └── <appId>
        ├── <depotId>_<manifestId>.manifest
        └── ...
```

Beispiel:

```text
API
├── Keys
│   └── 1091500.key
└── manifests
    └── 1091500
        ├── 1091501_123456789.manifest
        └── 1091506_987654321.manifest
```

## HTTP API

Alle Endpunkte sind `GET` Endpunkte.

### Authentifizierung

Du kannst den API-Key auf drei Arten senden.

Query-Parameter:

```text
http://localhost:8080/API/test?apiKey=dein-api-key
```

Header:

```text
X-API-Key: dein-api-key
```

Bearer Token:

```text
Authorization: Bearer dein-api-key
```

### Test

```text
GET /API/test
```

Erfolgreiche Antwort:

```text
server works
```

Beispiel:

```powershell
Invoke-WebRequest "http://localhost:8080/API/test?apiKey=dein-api-key"
```

### Depot-Key laden

```text
GET /API/keys/<appId>
```

Beispiel:

```powershell
Invoke-WebRequest "http://localhost:8080/API/keys/1091500?apiKey=dein-api-key" -OutFile "1091500.key"
```

Das Programm sucht zuerst exakt nach:

```text
API\Keys\<appId>.key
```

Falls diese Datei nicht existiert, wird eine `.key` Datei gesucht, deren Dateiname die AppId enthält.

### Manifeste laden

```text
GET /API/manifests/<appId>
```

Beispiel:

```powershell
Invoke-WebRequest "http://localhost:8080/API/manifests/1091500?apiKey=dein-api-key" -OutFile "1091500_manifests.zip"
```

Das Programm packt alle Dateien aus diesem Ordner in ein ZIP:

```text
API\manifests\<appId>\*.manifest
```

## Automatische Extraktion

Wenn `AutomaticExtractionEnabled` aktiv ist oder `--auto-extract` gesetzt wurde, versucht `TidesUpdater` regelmäßig, Keys und Manifeste über Steam zu aktualisieren.

Nach jedem automatischen Lauf zeigt die Konsole einen laufenden Countdown bis zur nächsten Extraktion an.

Die Konsole nutzt Farben für wichtige Zustände: erfolgreiche Aktionen grün, harte Fehler rot, überspringbare Fehler und Warnungen gelb/orange.

Die zu aktualisierenden Apps werden aus den vorhandenen `.key` Dateien erkannt:

```text
API\Keys\*.key
```

Für jede App wird eine Account-Datei erwartet:

```text
serversettings\accounts\<appId>.txt
```

Beispiel:

```text
serversettings\accounts\1091500.txt
```

Inhalt:

```text
steamusername:steampassword
```

Mehrere Accounts sind möglich, eine Zeile pro Account:

```text
account1:password1
account2:password2
```

Beim Programmstart werden alle konfigurierten Accounts einmal interaktiv geprüft. Wenn Steam Guard benötigt wird, kannst du den Code direkt in der Konsole eingeben. Erfolgreiche Logins werden mit `RememberPassword` gespeichert, damit spätere automatische Extraktionen möglichst ohne erneute Steam-Guard-Abfrage laufen.

Die gemerkten Steam-Sessions und Guard-Daten werden im Programmordner gespeichert:

```text
<Ordner der EXE>\account.config
```

Wenn ein gespeicherter Token abgelaufen ist, versucht `TidesUpdater` beim Startup-Check erneut eine Passwort-Anmeldung, sodass du Steam Guard wieder eingeben und die Session neu merken lassen kannst.

Wenn während der späteren automatischen Extraktion trotzdem unerwartet Steam Guard oder eine mobile Bestätigung benötigt wird, wird dieser Account für diesen Lauf übersprungen. Beim nächsten Programmstart kannst du ihn durch den interaktiven Startup-Check wieder vorbereiten.

Die extrahierten Daten werden automatisch in die API-Struktur geschrieben:

```text
API\Keys\<appId>.key
API\manifests\<appId>\*.manifest
```

## GitHub-Upload nach Extract

Wenn `GitHubUploadEnabled` aktiv ist oder `--github-upload` gesetzt wurde, lädt `TidesUpdater` nach jedem erfolgreichen Extract die erzeugten API-Dateien zusätzlich in ein GitHub-Repository.

Hochgeladen wird ins Repo unter `Manifests/<appId>/`. Die `.key` Datei und alle `.manifest` Dateien liegen dort zusammen in einem Ordner:

```text
Manifests/<appId>/<appId>.key
Manifests/<appId>/*.manifest
```

Pro App-Extract wird ein Commit erzeugt:

```text
Update Tides API files for AppId <appId>
```

Beispiel über Startparameter:

```powershell
$env:TIDESUPDATER_GITHUB_PAT = "github_pat_dein_token"

dotnet run --project .\TidesUpdater\TidesUpdater.csproj -- `
  --api-key "dein-api-key" `
  --auto-extract `
  --github-upload `
  --github-owner "dein-user" `
  --github-repo "dein-repo" `
  --github-branch "main"
```

Der PAT braucht für das Ziel-Repository mindestens:

```text
Contents: Read and write
Metadata: Read
```

Empfohlen ist ein fine-grained PAT, der nur Zugriff auf dieses eine Repository hat.

Wenn der GitHub-Upload fehlschlägt, bleibt der lokale Extract trotzdem erhalten. Das Programm schreibt den Fehler in die Konsole und läuft weiter.

## Typischer Ablauf

1. `settings.json` mit `ApiKey`, Port und optional Auto-Extraktion anlegen.
2. `API\Keys` und `API\manifests` bereitstellen oder durch automatische Extraktion erzeugen lassen.
3. Bei Auto-Extraktion pro App eine Datei unter `serversettings\accounts\<appId>.txt` anlegen.
4. Optional GitHub-Upload konfigurieren.
5. Programm starten.
6. API mit `/API/test` prüfen.
7. Clients können Keys und Manifeste über `/API/keys/<appId>` und `/API/manifests/<appId>` abrufen.

## Fehlerbilder

`API key is missing`

Der API-Key fehlt. Setze ihn in `settings.json` oder starte mit:

```powershell
--api-key "dein-api-key"
```

`Could not bind API server to all interfaces ... Falling back to localhost`

Das Programm konnte nicht auf `http://+:<port>/` lauschen und startet stattdessen auf `http://localhost:<port>/`. Für lokalen Betrieb ist das normalerweise okay.

`Key not found`

Für die angefragte AppId wurde keine passende Datei unter `API\Keys` gefunden.

`Manifest folder not found`

Für die angefragte AppId existiert kein Ordner unter `API\manifests\<appId>`.

`No manifest files found`

Der Manifest-Ordner existiert, enthält aber keine `.manifest` Dateien.

`GitHub upload enabled but incomplete`

GitHub-Upload ist aktiv, aber mindestens eine Einstellung fehlt: Owner, Repo, Branch oder Token.

`GitHub API failed`

GitHub hat den Upload abgelehnt. Häufige Ursachen sind ein falscher Token, fehlende `Contents: Read and write` Rechte, ein falscher Repo-Name oder ein nicht vorhandener Branch.
