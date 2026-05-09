# Pi-hole Auto-Backup mit n8n → SharePoint

Automatisierte tägliche Sicherung der Pi-hole v6 Konfiguration via n8n.
Backup wird per SSH auf dem Pi erstellt (Teleporter), abgeholt und in einen
SharePoint-Ordner hochgeladen. Telegram-Benachrichtigung als Statusmeldung.

## Inhalt

- [Architektur](#architektur)
- [Voraussetzungen](#voraussetzungen)
- [Setup](#setup)
  - [1. SSH-Key erzeugen](#1-ssh-key-auf-der-n8n-host-vm-erzeugen)
  - [2. Public Key übertragen](#2-public-key-auf-pi-hole-übertragen)
  - [3. SSH-Test](#3-ssh-test-key-basiert-ohne-passwort)
  - [4. Teleporter-Test](#4-teleporter-test-direkt-am-pi)
  - [5. Non-interaktiver Test](#5-non-interaktiver-test-von-der-n8n-vm)
  - [6. n8n SSH-Credential anlegen](#6-n8n-ssh-credential-anlegen)
- [Workflow-Knoten konfigurieren](#workflow-knoten-konfigurieren)
  - [Knoten 1: Cleanup](#knoten-1-cleanup-ssh-execute-command)
  - [Knoten 2: Teleporter ausführen](#knoten-2-teleporter-ausführen-ssh-execute-command)
  - [Knoten 3: Download](#knoten-3-download-ssh-file-download)
  - [Knoten 4: SharePoint Upload](#knoten-4-upload-zu-sharepoint-http-request)
  - [Knoten 5: Telegram](#knoten-5-telegram-benachrichtigung-optional)
  - [Schedule Trigger](#schedule-trigger)
- [Troubleshooting](#troubleshooting)
- [Sicherheit](#sicherheit)

## Architektur

```
┌──────────────┐   SSH    ┌────────────┐   SFTP    ┌──────────┐   REST API   ┌────────────┐
│ Schedule     │ ────────► │  Pi-hole   │ ─────────► │   n8n    │ ────────────► │ SharePoint │
│ Trigger      │          │ Teleporter │           │ Workflow │              │  (DMS)     │
└──────────────┘          └────────────┘           └──────────┘              └────────────┘
                                                        │
                                                        ▼
                                                  ┌──────────┐
                                                  │ Telegram │
                                                  └──────────┘
```

Workflow-Knoten in Reihenfolge:

1. **Schedule Trigger** — täglich, z. B. 03:00
2. **SSH Cleanup** — löscht Backups älter als 1 Tag aus `/tmp` auf dem Pi
3. **SSH Teleporter** — führt `sudo pihole-FTL --teleporter` aus
4. **SSH Download** — holt die ZIP-Datei vom Pi
5. **HTTP Request** — lädt Datei via SharePoint REST API hoch
6. **Telegram** — sendet Statusmeldung

## Voraussetzungen

- Pi-hole v6 (bei v5 ist der Befehl `pihole -a -t`, Pfad weicht ab)
- n8n (getestet auf 2.19.2, self-hosted Docker)
- SharePoint Online mit Microsoft Graph / SharePoint OAuth2 App-Registrierung
  und konfigurierter n8n-Credential vom Typ *Microsoft SharePoint OAuth2 API*
- Telegram Bot (optional)

## Setup

### 1. SSH-Key auf der n8n-Host-VM erzeugen

Dedizierter Key, ohne Passphrase (sonst läuft die Automation nicht
unbeaufsichtigt):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/n8n_pihole_backup -C "n8n-pihole-backup"
```

### 2. Public Key auf Pi-hole übertragen

```bash
ssh-copy-id -i ~/.ssh/n8n_pihole_backup.pub <USER>@<PI_IP>
```

`<USER>` ist der reguläre Pi-Benutzer (z. B. `pihole` oder `pi`).

### 3. SSH-Test (key-basiert, ohne Passwort)

```bash
ssh -i ~/.ssh/n8n_pihole_backup <USER>@<PI_IP>
```

### 4. Teleporter-Test direkt am Pi

```bash
sudo pihole-FTL --teleporter
```

Output ist ein Dateiname wie
`pi-hole_<host>_teleporter_<datum>_<zeit>_<tz>.zip` im aktuellen Verzeichnis.

> **Hinweis:** Bei Pi-hole v6 wurde der Teleporter-CLI-Befehl von
> `pihole -a -t` nach `pihole-FTL --teleporter` verschoben.

Falls `sudo` nach Passwort fragt: NOPASSWD-Eintrag in `/etc/sudoers.d/`
einrichten, sonst klemmt der Workflow.

### 5. Non-interaktiver Test von der n8n-VM

```bash
ssh -i ~/.ssh/n8n_pihole_backup <USER>@<PI_IP> "sudo pihole-FTL --teleporter"
```

Muss den Dateinamen ohne Passwortabfrage zurückliefern.

### 6. n8n SSH-Credential anlegen

UI: *Credentials → Create → SSH Private Key*

| Feld           | Wert                                      |
|----------------|-------------------------------------------|
| Name           | `pihole-backup-ssh`                       |
| Host           | `<PI_IP>`                                 |
| Port           | `22`                                      |
| Username       | `<USER>`                                  |
| Private Key    | Inhalt von `~/.ssh/n8n_pihole_backup`     |
| Passphrase     | leer                                      |

Den Private Key auf der n8n-VM ausgeben mit:

```bash
cat ~/.ssh/n8n_pihole_backup
```

Inkl. `-----BEGIN OPENSSH PRIVATE KEY-----` und `-----END...-----` kopieren.

## Workflow-Knoten konfigurieren

### Knoten 1: Cleanup (SSH Execute Command)

| Feld              | Wert                                                                            |
|-------------------|---------------------------------------------------------------------------------|
| Credential        | `pihole-backup-ssh`                                                             |
| Resource          | Command                                                                         |
| Command           | `find /tmp -maxdepth 1 -name 'pi-hole_*_teleporter_*.zip' -mtime +1 -delete`   |
| Working Directory | `/tmp`                                                                          |

Mit `-mtime +1` werden Backups älter als 24 h entfernt — es bleibt das aktuelle
und das von gestern erhalten (Generationsprinzip).

### Knoten 2: Teleporter ausführen (SSH Execute Command)

| Feld              | Wert                           |
|-------------------|--------------------------------|
| Credential        | `pihole-backup-ssh`            |
| Resource          | Command                        |
| Command           | `sudo pihole-FTL --teleporter` |
| Working Directory | `/tmp`                         |

`stdout` enthält den erzeugten Dateinamen.

### Knoten 3: Download (SSH File Download)

| Feld                | Wert                                      |
|---------------------|-------------------------------------------|
| Credential          | `pihole-backup-ssh`                       |
| Resource            | File                                      |
| Operation           | Download                                  |
| Path                | `/tmp/{{ $json.stdout.trim() }}`          |
| File Property Name  | `data`                                    |

Ergebnis: Binary-Item mit der ZIP-Datei.

### Knoten 4: Upload zu SharePoint (HTTP Request)

n8n's nativer SharePoint-Node zeigt im *From list*-Dropdown nur die
Default-Library (`Shared Documents`). Custom Libraries (z. B. `DMS`) sind
darüber **nicht** auswählbar — siehe
[n8n issue #20040](https://github.com/n8n-io/n8n/issues/20040). Saubere
Lösung: SharePoint REST API direkt via HTTP Request.

| Feld                  | Wert                                                                                                                                                                                  |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Method                | POST                                                                                                                                                                                  |
| URL (Expression!)     | `https://<TENANT>.sharepoint.com/sites/<SITE>/_api/web/GetFolderByServerRelativeUrl('/sites/<SITE>/<LIBRARY>/<FOLDER_PATH>')/Files/add(url='{{ $binary.data.fileName }}',overwrite=true)` |
| Authentication        | Predefined Credential Type                                                                                                                                                            |
| Credential Type       | Microsoft SharePoint OAuth2 API                                                                                                                                                       |
| Send Headers          | ON → Header `Accept: application/json;odata=verbose`                                                                                                                                  |
| Send Body             | ON → Body Content Type **Binary**, Input Field Name `data`                                                                                                                            |

> **Wichtig:** Das URL-Feld muss explizit auf **Expression** umgeschaltet sein,
> sonst wird `{{ $binary.data.fileName }}` als Literal behandelt und im
> Zielordner liegt eine Datei mit genau diesem Namen.

#### Beispiel-Mapping

| Platzhalter       | Beispielwert                                  |
|-------------------|-----------------------------------------------|
| `<TENANT>`        | `contoso`                                     |
| `<SITE>`          | `IT`                                          |
| `<LIBRARY>`       | `Documents` oder Custom Library z. B. `DMS`   |
| `<FOLDER_PATH>`   | `Backups/pihole`                              |

### Knoten 5: Telegram-Benachrichtigung (optional)

| Feld        | Wert                          |
|-------------|-------------------------------|
| Resource    | Message                       |
| Operation   | Send Text Message             |
| Chat ID     | `<TELEGRAM_CHAT_ID>`          |
| Parse Mode  | HTML                          |

Text:

```
✅ <b>Pi-hole Backup</b>
📄 <code>{{ $json.d.Name }}</code>
📦 {{ ($json.d.Length / 1024).toFixed(1) }} KB
📅 {{ $now.format('dd.MM.yyyy HH:mm') }}
```

> **Achtung MarkdownV2:** Bei Parse Mode `MarkdownV2` müssen `.`, `-`, `_`
> u. a. einzeln escaped werden — Dateinamen mit Zeitstempeln führen zu
> ständigen Parse-Fehlern. **HTML** umgeht das Problem.

### Schedule Trigger

z. B. täglich 03:00 (Cron `0 3 * * *`). Workflow danach **aktivieren**.

## Troubleshooting

| Symptom                                                              | Ursache / Lösung                                                                                                   |
|----------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| `No such file` beim SSH-Download                                     | `Working Directory` und `Path` müssen übereinstimmen. Bei v6 Teleporter wird die Datei im `cwd` abgelegt.         |
| `sudo: a password is required`                                       | NOPASSWD für `pihole-FTL` in `/etc/sudoers.d/` einrichten.                                                        |
| Datei in SharePoint heißt literal `{{ $binary.data.fileName }}`      | URL-Feld nicht im Expression-Modus.                                                                               |
| Custom Library taucht im SharePoint-Node nicht auf                   | Designschwäche des Nodes — HTTP Request mit REST API verwenden (siehe oben).                                      |
| Telegram: `Can't find end of the entity`                             | MarkdownV2 / Markdown beißt sich mit `_`/`-`/`.` im Dateinamen. Parse Mode auf **HTML** umstellen.                |
| Pi-hole v5 statt v6                                                  | Befehl ist `pihole -a -t /tmp/pihole-backup.tar.gz`, Dateierweiterung `.tar.gz` statt `.zip`.                     |

## Sicherheit

- Dedizierter SSH-Key nur für diesen Backup-Job (kein Wiederverwenden
  bestehender Keys).
- Auf dem Pi den Key auf den Backup-Befehl beschränken via `command="..."`
  in `~/.ssh/authorized_keys` (optional, härter):
  ```
  command="sudo pihole-FTL --teleporter",no-port-forwarding,no-agent-forwarding,no-X11-forwarding ssh-ed25519 AAAA... n8n-pihole-backup
  ```
  (Funktioniert nur sauber, wenn ausschließlich dieser eine Befehl benötigt
  wird — das Cleanup-Kommando muss dann in den `command=`-Wrapper integriert
  werden.)
- SharePoint OAuth2-App mit minimalen Scopes
  (`Sites.Selected` statt `Sites.ReadWrite.All`, falls möglich).
