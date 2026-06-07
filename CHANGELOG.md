# Changelog

Alle nennenswerten Änderungen an diesem Projekt werden in dieser Datei dokumentiert.

Das Format orientiert sich an [Keep a Changelog](https://keepachangelog.com/de/1.1.0/) und das Projekt folgt [Semantic Versioning](https://semver.org/lang/de/).

## [Unreleased]

### Added
- _Noch nichts._

### Changed
- _Noch nichts._

### Fixed
- _Noch nichts._

---

## [1.2.0] - 2026-06-07

### Changed
- Troubleshooting der `DeviceDisappeared` / `NewArray`-Phantom-Events grundlegend überarbeitet. Korrigierte Ursachenanalyse: Neueres mdadm exportiert bei Array-Namen mit Host-Präfix (`<hostname>:0`) kein `MD_DEVNAME`, daher legt die Debian-udev-Regel `63-md-raid-arrays.rules` den by-name-Symlink unter `/dev/md/` nicht an (bekannter OMV-8-Bug, openmediavault/openmediavault#2071). Neuer, downtime-freier Fix per lokaler udev-Regel (`99-local-md-symlink.rules`, Symlink aus dem Kernel-Namen `%k`) — ersetzt den bisherigen Ansatz über das Umschreiben der `ARRAY`-Zeile in `mdadm.conf` mit anschließendem Reboot.
- Troubleshooting-Datei stilistisch an die übrigen Dokumente angeglichen (Heading-Struktur mit `#`/`##`, Abschnitt „Verwandte Dokumente").

### Added
- Verlinkung der Phantom-Events-Anleitung in der README (Schnellnavigation und Feature-Liste „USB-RAID").

### Fixed
- Versionierung in Ordnung gebracht: Git-Tags `v1.0.0`–`v1.2.0` nachgezogen, sodass die zuvor ins Leere zeigenden Compare-/Release-Links am Dateiende funktionieren.
- Datumsformat der Version 1.0.0 auf ISO 8601 (`2026-04-23`) vereinheitlicht.

---

## [1.1.0] - 2026-05-07

### Added
- Troubleshooting-Abschnitt zu täglichen `DeviceDisappeared` / `NewArray`-Phantom-Events in den `cron.daily`-Mails: Ursachenanalyse (fehlender by-name-Symlink unter `/dev/md/`), POSIX-Stolperfalle bei Array-Namen mit Doppelpunkt, und Fix via Anpassung der `ARRAY`-Zeile in `mdadm.conf`.
- Hinweis zum häufigen `update-initramfs: command not found`-Fehler bei Wechsel zu root via `su` statt `su -` (PATH-Problem).

---

## [1.0.0] - 2026-04-23

### Added
- Erste Veröffentlichung der vollständigen Anleitung
- RAID 6 Aufbau auf USB-Platten mit Hotspares per `mdadm`-CLI
- Strategie zur Rollenverteilung der Platten anhand SMART-Werten
- Dauerhafte Konfiguration inkl. Initramfs-Update
- E-Mail-Benachrichtigungen für RAID- und SMART-Events
- Failover-Tests (Software und Hardware)
- Salt-State für update-sicheren Custom-`From`-Header in OMV-Notification-Mails

---

[Unreleased]: https://github.com/Phydran6/OMV-as-NAS/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/Phydran6/OMV-as-NAS/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/Phydran6/OMV-as-NAS/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/Phydran6/OMV-as-NAS/releases/tag/v1.0.0
