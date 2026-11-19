# NFSv4: Serverseitige Upload-Verarbeitung mit systemd

> Ergänzung zum NFSv4-Handbuch – Gespräch vom 19. November 2026, ab 08:57 Uhr.  
> Format: Markdown, zur Bearbeitung mit Pulsar oder vi/vim und zur Versionierung mit Git.

## Ziel und Umgebung

Auf dem NFSv4-Client werden Dateien in ein Upload-Verzeichnis geschrieben. Ein Dienst auf dem Server prüft regelmäßig, ob Dateien längere Zeit unverändert sind, und verschiebt sie in ein nicht über NFS freigegebenes Archiv.

| Rolle | Hostname | IP-Adresse |
|---|---|---|
| Server | `server1.srv.lcd` | `192.168.1.191` |
| Client | `client1.srv.lcd` | `192.168.1.223` |

| Zweck | Pfad |
|---|---|
| Upload auf dem Server | `/srv/nfs/praxis/upload/` |
| Archiv auf dem Server | `/srv/archiv/` |
| NFSv4-Mount auf dem Client | `/praxis/` |
| Upload auf dem Client | `/praxis/upload/` |

**Sicherheitsgrenze:** NFSv4 kennt keine `wo`-Mount-Option (write-only). Solange eine Datei im Uploadbereich liegt, kann der hochladende Client sie unter Umständen lesen, ändern oder löschen. Die folgende Lösung ist **eine zeitnahe Archivierung**, **kein garantiert sicherer Write-only-Dienst**.

## 1. Verzeichnisse vorbereiten (Server)

Auf `server1.srv.lcd`:

```bash
sudo mkdir -p /srv/nfs/praxis/upload /srv/archiv
sudo chown root:root /srv/nfs/praxis/upload
sudo chmod 1733 /srv/nfs/praxis/upload
sudo chown root:root /srv/archiv
sudo chmod 700 /srv/archiv
```

`1733` erlaubt anderen Benutzern das Schreiben und Betreten des Upload-Verzeichnisses, aber nicht dessen Auflistung. Das Sticky-Bit schränkt das Löschen fremder Dateien ein. **Es verhindert nicht, dass eine Datei mit bekanntem Namen gelesen werden kann, wenn deren Dateirechte das zulassen.** Die übergeordneten Verzeichnisse müssen für den Client passierbar sein.

Das Archiv `/srv/archiv` muss außerhalb aller über NFS exportierten Pfade liegen.

## 2. Übernahmeskript erstellen (Server)

```bash
sudo vim /usr/local/sbin/nfs-upload-archiv.sh
```

Beispiel für eine **einfache Testumgebung mit vertrauenswürdigen Clients**:

```bash
#!/usr/bin/env bash
set -euo pipefail

QUELLE=/srv/nfs/praxis/upload
ZIEL=/srv/archiv

# Keine parallelen Durchläufe.
exec 9>/run/nfs-upload-archiv.lock
flock -n 9 || exit 0

# Nur reguläre Dateien direkt im Upload-Verzeichnis.
# Dateien müssen seit mehr als zwei Minuten unverändert sein.
find "$QUELLE" -mindepth 1 -maxdepth 1 -type f -mmin +2 -print0 |
while IFS= read -r -d '' DATEI; do
    NAME=${DATEI##*/}

    # Vorhandene Archivnamen nicht überschreiben.
    if [[ -e "$ZIEL/$NAME" ]]; then
        logger -t nfs-upload "Archivname existiert bereits: $NAME"
        continue
    fi

    # Einfaches Lehrbeispiel; keine transaktionssichere Übernahme.
    if mv -n -- "$DATEI" "$ZIEL/$NAME"; then
        chmod 600 -- "$ZIEL/$NAME"
        chown root:root -- "$ZIEL/$NAME"
        logger -t nfs-upload "Archivierungsversuch abgeschlossen: $NAME"
    fi
done
```

```bash
sudo chown root:root /usr/local/sbin/nfs-upload-archiv.sh
sudo chmod 700 /usr/local/sbin/nfs-upload-archiv.sh
```

**Wichtige Einschränkungen dieses Beispiels:**

- `-mmin +2` bedeutet nicht, dass ein Upload vollständig abgeschlossen ist; eine Datei kann noch geöffnet sein.
- `mv -n` ist keine umfassende Absicherung gegen Rennen und Dateinamenkonflikte; je nach System kann ein bereits vorhandenes Ziel zum scheinbar erfolgreichen Aufruf führen. Prüfe daher nicht allein den Rückgabewert als Nachweis einer Übernahme.
- Liegen Quelle und Archiv auf unterschiedlichen Dateisystemen, kann `mv` intern kopieren und danach löschen; der Vorgang ist dann nicht atomar.
- Zwischen Verschieben und `chmod`/`chown` gibt es ein Zeitfenster. Für nicht vertrauenswürdige Clients oder sensible Daten ist das Skript **nicht ausreichend gehärtet**.
- Ein Client kann die Quelldatei während der Prüfung verändern. Für verlässliche Verarbeitung sind ein Upload-Abschlussprotokoll und ein geschützter Staging-Bereich erforderlich.

## 3. systemd-Service erstellen (Server)

```bash
sudo vim /etc/systemd/system/nfs-upload-archiv.service
```

```ini
[Unit]
Description=NFSv4 Upload-Dateien archivieren
After=local-fs.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/nfs-upload-archiv.sh
```

Der Service führt das Skript einmal aus und beendet sich danach.

## 4. systemd-Timer erstellen (Server)

```bash
sudo vim /etc/systemd/system/nfs-upload-archiv.timer
```

```ini
[Unit]
Description=NFSv4 Upload-Verzeichnis regelmaessig pruefen

[Timer]
OnBootSec=30s
OnUnitInactiveSec=15s
AccuracySec=1s
Unit=nfs-upload-archiv.service

[Install]
WantedBy=timers.target
```

Aktivieren:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nfs-upload-archiv.timer
systemctl status nfs-upload-archiv.timer
systemctl list-timers --all
```

Der Timer startet den Dienst etwa 15 Sekunden nach dem Ende des vorherigen Laufs; `AccuracySec=1s` begrenzt die zeitliche Zusammenfassung durch systemd. Die Mindestwartezeit für die Datei bleibt davon unabhängig bei mehr als zwei Minuten.

## 5. Funktionstest

Auf `client1.srv.lcd` (NFSv4-Freigabe muss bereits unter `/praxis` gemountet sein):

```bash
printf 'NFSv4 Upload Test\n' > /praxis/upload/test01.txt
```

Auf `server1.srv.lcd`:

```bash
sudo ls -l /srv/nfs/praxis/upload/
```

Nach mehr als zwei Minuten und dem nächsten Timerlauf:

```bash
sudo ls -l /srv/archiv/
```

Beispiel für den erwarteten Archivzustand:

```text
-rw------- 1 root root ... test01.txt
```

Auf dem Client sollte die Quelldatei nach erfolgreicher Übernahme nicht mehr vorhanden sein:

```bash
ls /praxis/upload/test01.txt
```

Bei Problemen prüfen, ob die Datei tatsächlich verschoben wurde und ob das Ziel existiert. Die Archivdatei ist absichtlich nicht über NFS erreichbar.

## 6. Protokollierung und manueller Lauf

```bash
sudo journalctl -u nfs-upload-archiv.service
sudo journalctl -t nfs-upload
sudo systemctl start nfs-upload-archiv.service
```

Auch beim manuellen Start berücksichtigt das Skript die Mindestwartezeit.

## 7. Warum nicht ausschließlich inotify?

`inotify` kann lokale Dateisystemereignisse wie `CLOSE_WRITE` melden. Bei Zugriffen über NFS darf jedoch nicht vorausgesetzt werden, dass das Schließen einer Datei durch einen entfernten Client auf dem Server immer als geeignetes `inotify`-Ereignis sichtbar wird. Deshalb verwendet dieses Beispiel einen **systemd-Timer mit periodischer Prüfung** statt einer ausschließlich ereignisgesteuerten Übernahme.

## 8. Verbesserung für zuverlässigere Uploads

Für einen geregelten Ablauf kann der Client zunächst unter einem temporären Namen schreiben und nach Abschluss umbenennen:

```text
bericht.part  ->  bericht.ready
```

Ein serverseitiger Dienst verarbeitet dann nur `.ready`-Dateien. Das Umbenennen signalisiert einen abgeschlossenen Upload **nur, wenn der Client das vereinbarte Protokoll korrekt einhält**. Bei nicht vertrauenswürdigen Clients ist es kein Sicherheitsnachweis.

Für produktive Umgebungen sollten zusätzlich umgesetzt werden:

1. Geschützter Staging-Bereich, auf den der NFS-Client keinen Zugriff hat.
2. Eindeutige, serverseitig vergebene Archivnamen.
3. Sichere Behandlung von Symlinks, Dateiaustausch und konkurrierenden Änderungen.
4. Übernahme erst nach einem überprüfbaren Upload-Abschluss.
5. Fehlerbehandlung und Protokollierung mit eindeutiger Erfolgsprüfung.
6. Bei echtem Upload-only-Bedarf: ein passend eingeschränkter SFTP- oder Anwendungsdienst statt eines direkt beschreibbaren NFS-Exports.

## 9. Zusammenfassung

- **NFSv4:** Freigabe und Transport der Dateien.
- **systemd-Timer:** Regelmäßiger Start der Verarbeitung.
- **Shell-Skript:** Einfache, zeitversetzte Archivierung für einen kontrollierten Testbetrieb.
- **Archiv:** Nicht über NFS freigegeben und nur für root zugänglich.
- **Sicherheitsziel:** Keine Garantie, dass der Client seine Dateien niemals lesen kann.

## 10. In Git mit Pulsar oder vim bearbeiten

Nach dem Herunterladen der Datei beispielsweise:

```bash
cd ~/NFS4book
cp ~/Downloads/NFSv4-Serverseitige-Upload-Verarbeitung.md .
pulsar NFSv4-Serverseitige-Upload-Verarbeitung.md
# alternativ:
vim NFSv4-Serverseitige-Upload-Verarbeitung.md
```

Anschließend:

```bash
git add NFSv4-Serverseitige-Upload-Verarbeitung.md
git commit -m "NFSv4: serverseitige Upload-Verarbeitung dokumentiert"
git push origin main
```

> Den Branch-Namen `main` bei Bedarf an deinen tatsächlichen Git-Branch anpassen. Das Dokument ist eine eigenständige Ergänzung und überschreibt keine vorhandenen Kapitel.
