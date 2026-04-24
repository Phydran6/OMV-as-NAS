# OpenMediaVault: RAID 6 mit Hotspares auf USB-Platten — Vollständige Anleitung

Eine Schritt-für-Schritt-Anleitung zum Aufsetzen eines Software-RAID 6 mit mehreren Hotspares auf USB-angeschlossenen Festplatten unter OpenMediaVault (OMV). Enthält den Umweg um die bewusste Einschränkung der OMV-Web-UI, die USB-Platten beim RAID-Erstellen und beim Hotspare-Hinzufügen ausblendet, sowie eine dauerhafte Anpassung des E-Mail-Absenders für Benachrichtigungen.

## Inhaltsverzeichnis

- [Ausgangslage und Ziel](#ausgangslage-und-ziel)
- [Warum die CLI nötig ist](#warum-die-cli-nötig-ist)
- [Strategie: Rollenverteilung der Platten](#strategie-rollenverteilung-der-platten)
- [Voraussetzungen](#voraussetzungen)
- [Teil 1: RAID-Aufbau](#teil-1-raid-aufbau)
  - [1. Aktuellen Zustand prüfen](#1-aktuellen-zustand-prüfen)
  - [2. Platten identifizieren](#2-platten-identifizieren)
  - [3. SMART-Werte prüfen](#3-smart-werte-prüfen)
  - [4. Alle Platten sauber wipen](#4-alle-platten-sauber-wipen)
  - [5. RAID 6 mit aktiven Platten erstellen](#5-raid-6-mit-aktiven-platten-erstellen)
  - [6. Sync-Fortschritt kontrollieren](#6-sync-fortschritt-kontrollieren)
  - [7. Hotspares hinzufügen](#7-hotspares-hinzufügen)
  - [8. Konfiguration persistieren](#8-konfiguration-persistieren)
- [Teil 2: Dateisystem und Freigaben](#teil-2-dateisystem-und-freigaben)
- [Teil 3: Benachrichtigungen einrichten](#teil-3-benachrichtigungen-einrichten)
- [Teil 4: Failover-Test](#teil-4-failover-test)
- [Teil 5: Mail-Absender dauerhaft ändern (Salt-State)](#teil-5-mail-absender-dauerhaft-ändern-salt-state)
- [Wartung und Troubleshooting](#wartung-und-troubleshooting)
- [Wichtiger Hinweis](#wichtiger-hinweis)

---

## Ausgangslage und Ziel

- Mehrere USB-angeschlossene Festplatten (in diesem Beispiel 6 × ~1,8 TB), Bezeichnung `/dev/sda` bis `/dev/sdf`
- Boot läuft von einer separaten NVMe
- Zielkonfiguration: **RAID 6 mit 4 aktiven Platten und 2 Hotspares**
- Damit verträgt das Array bis zu 4 sequenzielle Plattenausfälle, bevor Daten verloren gehen: 2 gleichzeitige durch die doppelte Parität von RAID 6, anschließend 2 weitere Rebuilds über die Spares

Die Anleitung lässt sich problemlos auf andere Plattenzahlen übertragen — die Logik bleibt gleich.

## Warum die CLI nötig ist

OMV blockiert in der Web-UI bewusst sowohl das RAID-Erstellen als auch das Hinzufügen von Hotspares für USB-Geräte. In der GUI erscheint der Hinweis:

> *Devices connected via USB will not be listed (too unreliable).*

Das ist kein Bug, sondern Absicht der Entwickler. USB-Bridges sind oft unzuverlässig (Spin-Down, vertauschte Serials, Reconnect-Probleme), und zu viele Anwender hatten damit Datenverlust. Die Sperre lässt sich aber umgehen, indem das Array per `mdadm` auf der Kommandozeile erzeugt wird. OMV erkennt das fertige Array danach automatisch und verwaltet es in der GUI wie jedes andere.

## Strategie: Rollenverteilung der Platten

Wenn nicht alle Platten gleich gesund sind, lohnt es sich die Rollen bewusst zu verteilen:

- **Platten mit verdächtigen SMART-Werten → als aktive Mitglieder** einsetzen
- **Die gesündesten Platten → als Hotspares** zurückhalten

Hintergrund: Spares drehen sich zwar mit, werden aber nicht beschrieben. Fällt eine aktive Platte aus, übernimmt ein Spare den Rebuild — dieser Rebuild ist die stressigste Phase für die beteiligten Platten. Eine angeschlagene Platte als Spare würde genau dann einspringen, wenn es am meisten weh tut, wenn sie selbst versagt. Deshalb gehören die stabilsten Platten auf die Spare-Position.

Relevante SMART-Attribute zum Bewerten:

- `Reallocated_Sector_Ct` (5)
- `Current_Pending_Sector` (197)
- `Offline_Uncorrectable` (198)
- `Reported_Uncorrect` (187)
- `UDMA_CRC_Error_Count` (199)
- `Power_On_Hours` (9)
- `Start_Stop_Count` (4)
- `High_Fly_Writes` (189, bei Seagate-Platten aussagekräftig)

Faustregel: Alles was bei Realloc / Pending / Offline_Uncorrectable / Reported_Uncorrect über Null geht, ist ein Warnsignal. Hohe Power-On-Hours und Start/Stop-Counts allein ohne Fehlerzähler sind kein Ausschlussgrund, erhöhen aber das Risiko.

## Voraussetzungen

- OMV mit SSH-Zugang als `root` (oder sudo-fähiger User)
- `mdadm` ist auf OMV-Systemen vorinstalliert
- `smartmontools` für die SMART-Analyse (meist ebenfalls vorhanden; bei Bedarf nachinstallieren mit `apt install smartmontools`)
- **Backup!** RAID ersetzt kein Backup. Alle Daten auf den Platten gehen beim Wipen verloren.

---

## Teil 1: RAID-Aufbau

### 1. Aktuellen Zustand prüfen

```bash
cat /proc/mdstat
```

Falls noch ein altes oder inaktives Array existiert (z. B. `md127`), erst stoppen:

```bash
mdadm --stop /dev/md127
```

Danach `cat /proc/mdstat` erneut ausführen und sicherstellen, dass kein Array mehr aktiv ist.

### 2. Platten identifizieren

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL
```

Seriennummern notieren. Bei einem Defekt kann man so die richtige Platte physisch identifizieren, selbst wenn sich durch Umstecken die Device-Namen geändert haben.

### 3. SMART-Werte prüfen

Für jede Platte einzeln:

```bash
smartctl -A /dev/sdX
```

Anhand der Werte die Rangfolge festlegen: welche Platten werden aktiv, welche werden Spares. Die gesündesten gehören auf die Spare-Position.

### 4. Alle Platten sauber wipen

Entfernt Filesystem-Signaturen **und** alte RAID-Superblocks:

```bash
for d in sda sdb sdc sdd sde sdf; do
  echo "=== Wiping /dev/$d ==="
  wipefs -a /dev/$d
  mdadm --zero-superblock /dev/$d 2>/dev/null || true
done
```

Verifizieren, dass wirklich nichts mehr übrig ist:

```bash
for d in sda sdb sdc sdd sde sdf; do
  echo "=== /dev/$d ==="
  mdadm --examine /dev/$d 2>&1 | head -5
done
```

Erwartet wird pro Platte: `mdadm: No md superblock detected on /dev/sdX.`

### 5. RAID 6 mit aktiven Platten erstellen

Die Reihenfolge der Geräte in diesem Befehl bestimmt die Rolle: diese Platten werden aktive Mitglieder. Im Beispiel 4 Platten als aktives Set:

```bash
mdadm --create /dev/md0 --level=6 --raid-devices=4 --bitmap=internal \
  /dev/sda /dev/sdb /dev/sde /dev/sdc
```

Die Write-Intent-Bitmap (`--bitmap=internal`) beschleunigt spätere Rebuilds erheblich, weil nur die als "dirty" markierten Bereiche neu geschrieben werden müssen statt der gesamten Platte.

### 6. Sync-Fortschritt kontrollieren

```bash
cat /proc/mdstat
```

Erwartete Ausgabe in etwa:

```
md0 : active raid6 sdc[3] sde[2] sdb[1] sda[0]
      3906764800 blocks super 1.2 level 6, 512k chunk, algorithm 2 [4/4] [UUUU]
      [>....................]  resync =  0.1% (...) finish=753min speed=43000K/sec
      bitmap: 15/15 pages [60KB], 65536KB chunk
```

Wichtig ist `[4/4] [UUUU]` — alle aktiven Platten sind da. Der Initial-Resync dauert je nach Geschwindigkeit mehrere Stunden bis etwa einen halben Tag. Das Array ist während des Resyncs bereits voll nutzbar.

### 7. Hotspares hinzufügen

Da das Array bereits komplett ist, werden die beiden neuen Platten automatisch als Spares eingetragen:

```bash
mdadm --add /dev/md0 /dev/sdd /dev/sdf
```

Prüfen:

```bash
cat /proc/mdstat
```

Jetzt sollte in der Array-Zeile das `(S)` hinter den Spare-Platten erscheinen:

```
md0 : active raid6 sdf[5](S) sdd[4](S) sdc[3] sde[2] sdb[1] sda[0]
```

### 8. Konfiguration persistieren

Damit das Array nach einem Reboot zuverlässig als `/dev/md0` (und nicht unter einem generischen Namen wie `md127`) wieder zusammengebaut wird, wird eine ARRAY-Zeile in die `mdadm.conf` geschrieben:

```bash
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

Dann die Datei kontrollieren:

```bash
cat /etc/mdadm/mdadm.conf
```

Erwartet wird genau **eine** `ARRAY`-Zeile. Falls dort mehrere stehen (z. B. vom alten Array), die veralteten Einträge entfernen.

Zum Schluss das Initramfs neu bauen, damit der Kernel die Config bereits beim Boot kennt:

```bash
update-initramfs -u
```

---

## Teil 2: Dateisystem und Freigaben

Das Dateisystem wird **nicht** per CLI angelegt, sondern über die OMV-Oberfläche — so kennt OMV das Filesystem direkt und kann Mountpoint, Freigaben und Quotas verwalten:

- **Datenspeicher → Dateisysteme → Erstellen und mounten**
- Gerät: `/dev/md0`
- Typ: `ext4` (solide und unkompliziert) oder `xfs` (bessere Performance bei großen Dateien und paralleler I/O)

Das Filesystem kann direkt erstellt werden — der Resync läuft im Hintergrund weiter und muss nicht abgewartet werden.

Anschließend in der GUI wie gewohnt Freigaben einrichten (SMB, NFS, etc.).

---

## Teil 3: Benachrichtigungen einrichten

Damit der Ausfall einer Platte nicht unbemerkt bleibt:

- **System → Benachrichtigung → Einstellungen** → SMTP-Zugangsdaten eintragen, Empfängeradresse setzen, Testmail senden
- Im Reiter **Benachrichtigungen** aktivieren:
  - `mdadm` (RAID-Events: degraded, failed, spare activated)
  - `S.M.A.R.T.` (frühzeitige Plattenwarnungen)
- Unter **Datenspeicher → S.M.A.R.T.** den Dienst aktivieren und für jede Platte die Überwachung einschalten; regelmäßige Short-/Long-Selftests einplanen

Bei Gmail als SMTP-Relay ist zwingend ein **App-Passwort** nötig (Google hat die Option "Less Secure Apps" im Mai 2022 entfernt). Zweiten Faktor aktivieren, App-Passwort erzeugen, in OMV eintragen.

---

## Teil 4: Failover-Test

Nach abgeschlossenem Resync empfehlenswert: einen Ausfall simulieren, um zu verifizieren dass der Hotspare automatisch einspringt und dass die Benachrichtigungskette funktioniert.

### Sanfte Variante (Software-Fail)

```bash
mdadm --manage /dev/md0 --fail /dev/sdb
```

`cat /proc/mdstat` sollte danach zeigen, dass ein Spare aktiv wird und ein Rebuild startet. Die fehlerhaft markierte Platte kann danach wieder integriert werden:

```bash
mdadm --manage /dev/md0 --remove /dev/sdb
mdadm --manage /dev/md0 --add /dev/sdb
```

Sie wird als neuer Spare eingereiht.

### Harte Variante (Hardware-Fail)

USB-Kabel einer Platte im laufenden Betrieb ziehen. Das testet zusätzlich das Verhalten der USB-Bridge und die gesamte Benachrichtigungskette unter realistischen Bedingungen.

Wichtig zu wissen: **Der Rebuild-Prozess läuft vollautomatisch auf mdadm-Ebene im Kernel.** OMV ist dabei nur Beobachter und zeigt den Status an. Es ist kein manuelles Eingreifen nötig. Sobald der Rebuild fertig ist, ist die ehemalige Spare ein vollwertiges aktives Mitglied, das Array hat dann allerdings einen Spare weniger.

---

## Teil 5: Mail-Absender dauerhaft ändern (Salt-State)

Standardmäßig schickt OMV Benachrichtigungen mit einem technischen Absender wie `jo@wasup.com` oder `root`. Für bessere Übersichtlichkeit im Postfach ist ein sprechender Display-Name wünschenswert, z. B. `NAS <jo@wasup.com>`.

### Warum es in der GUI nicht direkt geht

Das Feld "Sender email" in der OMV-Notification-GUI validiert strikt als reine E-Mail-Adresse. Ein Display-Name mit spitzen Klammern wird als ungültig abgelehnt.

### Warum direktes Editieren nicht reicht

`/etc/postfix/smtp_header_checks` wird von OMV über einen Salt-State (`/srv/salt/omv/deploy/postfix/50smtp_header_checks.sls`) verwaltet. Bei jedem UI-Apply wird die Datei mit `file.managed` komplett neu geschrieben — manuelle Änderungen gehen dabei verloren.

### Die saubere Lösung: eigener Salt-State

Die OMV-Hauptentwickler empfehlen für dauerhafte Zusatzkonfigurationen einen eigenen Salt-State mit höherer Nummer als die OMV-State-Dateien (`50...` → eigener `99...`), der `file.append` statt `file.managed` nutzt. Diese eigenen Dateien werden von OMV-Updates nicht angefasst.

**Schritt 1:** Test, ob der SMTP-Provider den Display-Name durchreicht:

```bash
cp /etc/postfix/smtp_header_checks /etc/postfix/smtp_header_checks.bak
echo '/^From:/ REPLACE From: NAS <jo@wasup.com>' >> /etc/postfix/smtp_header_checks
systemctl reload postfix
```

Dann in der OMV-GUI eine Test-Mail schicken und im Posteingang prüfen, was als Absender ankommt. Gmail z. B. lässt den Display-Namen in der Regel durch, kann ihn aber unter Umständen durch den im Google-Konto hinterlegten Namen ersetzen.

Wenn der Display-Name durchkommt: weiter mit Schritt 2. Falls nicht: zurückrollen mit

```bash
cp /etc/postfix/smtp_header_checks.bak /etc/postfix/smtp_header_checks
systemctl reload postfix
```

**Schritt 2:** Eigenen Salt-State anlegen, der die Zeile nach jedem OMV-Apply wieder anhängt:

```bash
cat > /srv/salt/omv/deploy/postfix/99custom_from_header.sls << 'EOF'
# Custom: rewrite From header to show display name.
# This file is NOT part of OpenMediaVault and persists across UI applies.
configure_postfix_custom_from_header:
  file.append:
    - name: "/etc/postfix/smtp_header_checks"
    - text: "/^From:/ REPLACE From: NAS <jo@wasup.com>"
EOF
```

**Schritt 3:** Postfix-Deploy ausführen, um die Konfiguration zu aktivieren:

```bash
/usr/sbin/omv-salt deploy run postfix
```

Wichtige Zeilen im Output:

- `configure_postfix_smtp_header_checks ... File ... updated` → OMV schreibt die Datei neu und entfernt dabei die manuelle Zeile
- `configure_postfix_custom_from_header ... Appended 1 lines` → der eigene State hängt die Zeile wieder an

Der eventuelle Fehler `Command 'postfix' failed with return code: 127` bei `postfix_set_permissions` ist ein bekanntes PATH-Problem der Salt-Umgebung unter OMV, unabhängig von dieser Änderung und unkritisch. Die Permissions werden bei jedem Postfix-Start automatisch gesetzt.

**Schritt 4:** Endergebnis kontrollieren:

```bash
cat /etc/postfix/smtp_header_checks
```

Erwartet wird:

```
# Append the hostname to the email subject.
/^Subject: (.*)/ REPLACE Subject: [nas2.phytech.de] ${1}
/^From:/ REPLACE From: NAS <jo@wasup.com>
```

Die Lösung ist **idempotent** (keine Duplikate bei wiederholter Ausführung) und **update-sicher** (eigene Dateien in `/srv/salt/omv/deploy/` werden von OMV-Updates nicht überschrieben).

### Anpassung an eigene Werte

Wenn sich Absendername oder -adresse ändern, einfach die Zeile in `/srv/salt/omv/deploy/postfix/99custom_from_header.sls` editieren und `/usr/sbin/omv-salt deploy run postfix` erneut ausführen.

---

## Wartung und Troubleshooting

### RAID-Status prüfen

```bash
cat /proc/mdstat
mdadm --detail /dev/md0
```

### Platte manuell als defekt markieren

```bash
mdadm --manage /dev/md0 --fail /dev/sdX
mdadm --manage /dev/md0 --remove /dev/sdX
```

### Neue Platte einbinden

Nachdem eine defekte Platte physisch getauscht wurde:

```bash
wipefs -a /dev/sdX
mdadm --zero-superblock /dev/sdX
mdadm --add /dev/md0 /dev/sdX
```

Die Platte wird automatisch als Spare eingereiht oder (falls das Array degraded ist) direkt für den Rebuild verwendet.

### SparesMissing-Meldung

Tritt nach einem Rebuild auf, wenn mdadm denkt, die Spare-Zahl sei kleiner als konfiguriert. Abhilfe: `/etc/mdadm/mdadm.conf` öffnen und `spares=N` auf die tatsächliche Spare-Zahl anpassen, oder die Zeile ganz entfernen, dann `update-initramfs -u`.

### Resync zu langsam

Kernel-Default ist für USB-Bridges oft zu konservativ. Temporär erhöhen:

```bash
echo 200000 > /proc/sys/dev/raid/speed_limit_min
echo 500000 > /proc/sys/dev/raid/speed_limit_max
```

Dauerhaft in `/etc/sysctl.d/raid.conf` eintragen.

---

## Wichtiger Hinweis

RAID ist Ausfallsicherheit, **kein Backup**. Es schützt gegen Hardware-Ausfall einzelner Platten, aber nicht gegen:

- versehentliches Löschen oder Überschreiben
- Ransomware und andere Malware
- Dateisystem-Korruption
- Diebstahl, Brand, Überspannung
- Bedienfehler

Für echte Datensicherheit braucht es zusätzlich ein separates Backup, idealerweise an einem anderen Ort.
