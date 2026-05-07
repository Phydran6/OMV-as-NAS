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

## [1.1.0] - 2026-05-07

### Added
- Troubleshooting-Abschnitt zu täglichen `DeviceDisappeared` / `NewArray`-Phantom-Events in den `cron.daily`-Mails: Ursachenanalyse (fehlender by-name-Symlink unter `/dev/md/`), POSIX-Stolperfalle bei Array-Namen mit Doppelpunkt, und Fix via Anpassung der `ARRAY`-Zeile in `mdadm.conf`.
- Hinweis zum häufigen `update-initramfs: command not found`-Fehler bei Wechsel zu root via `su` statt `su -` (PATH-Problem).

---

## [1.0.0] -  Apr 23, 2026

### Added
- Erste Veröffentlichung der vollständigen Anleitung
- RAID 6 Aufbau auf USB-Platten mit Hotspares per `mdadm`-CLI
- Strategie zur Rollenverteilung der Platten anhand SMART-Werten
- Dauerhafte Konfiguration inkl. Initramfs-Update
- E-Mail-Benachrichtigungen für RAID- und SMART-Events
- Failover-Tests (Software und Hardware)
- Salt-State für update-sicheren Custom-`From`-Header in OMV-Notification-Mails

---

[Unreleased]: https://github.com/Phydran6/OMV-as-NAS/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/Phydran6/OMV-as-NAS/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/Phydran6/OMV-as-NAS/releases/tag/v1.0.0
