# Praxiseigener Git-Updateablauf (Kienzlefon-App)

Stand: November 2026. Dieser Ablauf gilt für die **Kienzlefon-VM** mit
`/home/USER/kienzlefon-app` als Git-Checkout und der produktiven
`/usr/share/kienzlefon-app/telepraxis-app.php`.

## Architektur

- Der Git-Checkout ist **nicht** der produktive Webroot. Produktionsdateien,
  Inbox, Zugangsdaten und Aufnahmen gehören nicht ins Git-Repository.
- Produktive App: `/usr/share/kienzlefon-app/telepraxis-app.php`.
- Eingänge: `/srv/telepraxis/KANAL/inbox` (konfigurierter Pfad, nicht
  `/usr/share/kienzlefon-app/inbox`).
- WAVs: `/usr/share/kienzlefon-app/recordings` ist ein Webserver-Symlink auf
  `/var/spool/asterisk/kienzlefon/ready`.
- Middleware: `/opt/kienzlefon-t2med`, Config: `/etc/kienzlefon-t2med/config.toml`.
  **Wird mit diesem App-Updater niemals aktualisiert.**

## Warum v1.1 mit `--app-only`?

`kienzlefon-app-update-v1.0.sh` setzt eine vollständige SMS-Installation
einschließlich `sms-config.php` voraus. In dieser Praxisinstallation existieren
produktive `telepraxis-app.php` und `telepraxis-sms.php`, aber **keine**
`sms-config.php`. V1.1 belässt den bisherigen vollständigen Updateweg,
ergänzt jedoch den **App-only-Modus**, der *ausschließlich*
`telepraxis-app.php` austauscht und alle übrigen Dateien und Dienste in Ruhe lässt
(abgesehen vom kurzen Stop/Start von PHP-FPM für die Aktualisierung).

### Automatisch übernommene Werte

Folgende bestehende PHP-Konstanten werden über den PHP-Tokenizer ermittelt;
die neuen Quellwerte werden **nur für bekannte Konfigurationskonstanten**
durch die bisherigen Werte ersetzt, nicht durch ein globales Suchen/Ersetzen:

- Pflichtkonstanten: `TELEPRAXIS_INBOX_DIR`, `TELEPRAXIS_ADMIN_PASSWORD`,
  `TELEPRAXIS_POLL_INTERVAL_MS`, `TELEPRAXIS_DEFAULT_TIMEZONE`,
  `TELEPRAXIS_WORKPLACE_MAXLEN`
- Versionsabhängige Konstanten: `TELEPRAXIS_USER_MAXLEN`,
  `TELEPRAXIS_T2MED_ENABLED`, `TELEPRAXIS_T2MED_BASE_URL`,
  `TELEPRAXIS_RECORDINGS_BASE_URL`

Eine versionsabhängige Konstante wird übernommen, wenn sie schon in der
installierten Datei existiert. **Hat die vorhandene App eine dieser Konstanten,
fehlt sie aber im neuen Quellstand, bricht der Updater ab**, anstatt sie still
zu verwerfen. Wenn die Konstante vorher fehlte, gilt der neue Standardwert
aus dem Repository.

Lokale Codeänderungen *außerhalb* dieser Werte werden **nicht** automatisch
übernommen. Vor Updates daher einen geprüften, sauberen Quellbranch verwenden.

## Einmalige Vorbereitung

Als Benutzer `philip` im Git-Checkout:

```bash
cd /home/USER/kienzlefon-app
git fetch origin
git switch --track origin/feature/config-preserving-app-updater
git status --short --branch
```

Falls der Branch schon lokal existiert, stattdessen
`git switch feature/config-preserving-app-updater` und `git pull --ff-only`.

Der Branch baut auf `integration/auto-close-recording-miniplayer` auf:
**Auto-Close und Miniplayer sind beide enthalten.**

## Vor jedem Update

```bash
cd /home/USER/kienzlefon-app
git pull --ff-only
git status --short --branch  # keine lokalen Änderungen
bash -n kienzlefon-app-update-v1.1.sh
php -l telepraxis-app.php
python3 -B -m unittest discover -s tests -p 'test_app_update.py'
```

### Dry-run (ändert keine produktiven Dateien, Dienste oder Sicherungen)

```bash
sudo bash /home/USER/kienzlefon-app/kienzlefon-app-update-v1.1.sh \
  --webroot /usr/share/kienzlefon-app \
  --source-dir /home/USER/kienzlefon-app \
  --app-only --check
```

Der Updater benötigt im App-only-Modus zwingend `--source-dir`, damit nicht
versehentlich eine fremde/ältere Online-Version statt des geprüften Git-Checkouts
verwendet wird.

### Produktives Update mit Bestätigung

```bash
sudo bash /home/USER/kienzlefon-app/kienzlefon-app-update-v1.1.sh \
  --webroot /usr/share/kienzlefon-app \
  --source-dir /home/USER/kienzlefon-app \
  --app-only \
  --web-service php8.4-fpm.service
```

Das Skript zeigt zunächst den Plan und fragt nach Bestätigung.
Es erstellt ein privates Backup unter `/var/backups/kienzlefon-app/`,
stoppt PHP-FPM bei zuvor aktivem Dienst, ersetzt die Datei atomar und startet
den Dienst anschließend neu. Tritt ein Installations-/Dienstfehler auf,
versucht das Skript automatisch den Rollback und stellt den vorherigen
Dienstzustand wieder her. Die Kienzlefon-Telefonie wird nicht angehalten;
webbasierte PHP-Oberflächen sind jedoch während des kurzen FPM-Neustarts nicht
verfügbar.

**Nicht** `--no-service` verwenden, solange PHP-FPM läuft.
**Nicht** `--yes` verwenden, wenn du die interaktive Freigabe behalten möchtest.

## Nach dem Update

```bash
sudo php -l /usr/share/kienzlefon-app/telepraxis-app.php
sudo systemctl is-active php8.4-fpm
sudo grep -E '^const TELEPRAXIS_(INBOX_DIR|T2MED_ENABLED|T2MED_BASE_URL|RECORDINGS_BASE_URL) =' \
  /usr/share/kienzlefon-app/telepraxis-app.php
```

Beim Teilen der Ausgaben keine Passwörter/Tokens mitsenden. Browser testen:
App auf `https://KIENZLEFON-HOST/`, Miniplayer, Rezeptbestellung mit mehreren
WAV-Segmenten, Polling ohne Abspielunterbrechung sowie T2med-Auto-Close.

## Fehler und manueller Rollback

Bei `--check`-Fehlern wird keine Produktivdatei verändert. Bei Fehlern während
des Updates meldet das Skript den Zustand des automatischen Rollbacks.
Vor eigenständigen Eingriffen dessen Meldung prüfen.

Das Backup ist außerhalb des Webroots und nur für root zugänglich:

```bash
sudo find /var/backups/kienzlefon-app -maxdepth 2 -name manifest.json -print
```

Für einen *bewusst gewählten*, vorhandenen Backupordner `BACKUP` (zuvor anhand
Manifest und Zeitpunkt prüfen):

```bash
BACKUP=/var/backups/kienzlefon-app/GEPRUEFTER_BACKUPORDNER
sudo php -l "$BACKUP/web/telepraxis-app.php"
sudo install -o root -g root -m 0644 "$BACKUP/web/telepraxis-app.php" \
  /usr/share/kienzlefon-app/.telepraxis-app.php.rollback
sudo mv -f /usr/share/kienzlefon-app/.telepraxis-app.php.rollback \
  /usr/share/kienzlefon-app/telepraxis-app.php
sudo systemctl reload php8.4-fpm
```

## Abgrenzung

Diese Änderung automatisiert **keine** selbsttätigen Hintergrund-Updates.
Sie aktualisiert **nicht** die unabhängige `kienzlefon-t2med`-Middleware,
`telepraxis-sms.php`, die Inbox, den Spool, Nginx oder die ACLs.
Es ist auch keine Datenmigration oder neue Audio-Aufbewahrungslogik enthalten.
