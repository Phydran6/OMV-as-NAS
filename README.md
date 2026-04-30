# OMV USB-RAID 6 mit Hotspares + ACPI-Fixes für alte Hardware

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenMediaVault](https://img.shields.io/badge/OpenMediaVault-7%2F8-blue)](https://www.openmediavault.org/)
[![Platform: Debian](https://img.shields.io/badge/platform-Debian-red)](https://www.debian.org/)

Eine komplette, praxiserprobte Sammlung von Anleitungen rund um OpenMediaVault (OMV) auf günstiger oder älterer Hardware — inklusive Lösungen für Probleme, die in der OMV-GUI oder den offiziellen Dokumentationen nicht abgedeckt sind.

## Was dieses Repo abdeckt

### USB-RAID
- Software-RAID 6 mit mehreren Hotspares per `mdadm` (CLI)
- Strategische Rollenverteilung der Platten nach SMART-Werten
- Dauerhafte Konfiguration inklusive Initramfs-Update
- E-Mail-Benachrichtigungen bei Plattenausfällen
- Failover-Tests (Software und Hardware)
- Dauerhafter Custom-`From`-Header für Notification-Mails via Salt-State (überlebt OMV-Updates)

### OMV auf alter Hardware
- OMV 7 Installation auf 2010er-Atom-Boards (Cedarview & Co.)
- BIOS-only Boot, kein UEFI
- Versionswahl bei Kernel-Inkompatibilität (OMV 8/Kernel 6.12 zu neu für alte Chipsätze)

### ACPI-BIOS-Bugs fixen (DSDT-Override)
- Shutdown-Hänger mit `_PTS` / TPM-Fehlern reparieren
- Falsche `_S5`-States korrigieren
- DSDT dumpen, patchen, neu kompilieren und per Initramfs einbinden

## Schnellnavigation

- [USB-RAID Setup](docs/setup.md)
- [Mail-Absender anpassen](docs/setup.md#mail-absender-dauerhaft-ändern-salt-state)
- [OMV auf alter Hardware]([docs/omv-on-old-hardware.md](https://github.com/Phydran6/OMV-as-NAS/blob/main/omv-on-old-hardware.md))
- [ACPI Shutdown Fix (DSDT-Override)](docs/acpi-shutdown-fix.md)
- [Lizenz](LICENSE)

## Zielgruppe

Admins und Homelab-Enthusiasten, die:

- ein OMV-NAS mit USB-Festplatten betreiben (oder planen)
- ein RAID mit maximaler Ausfallsicherheit wollen
- nicht auf die OMV-GUI beschränkt sein möchten, wenn diese USB-Geräte blockiert
- ihre Konfiguration dauerhaft und update-sicher gestalten wollen
- alte NAS-Hardware (Thecus, Wortmann TERRA, QNAP TS-Serie etc.) mit aktuellem Linux/OMV weiternutzen möchten
- mit ACPI-BIOS-Bugs auf End-of-Life-Hardware konfrontiert sind

## Warum USB + RAID?

Kurz: Weil es geht und es für viele Setups die günstigste Lösung ist. Lang: USB-Bridges gelten zwar als unzuverlässiger als direkte SATA/SAS-Anbindung (Spin-Down, Reconnect-Probleme, gelegentlich vertauschte Serials), aber mit der richtigen RAID-Konfiguration (RAID 6 + Hotspares) und angemessenen Benachrichtigungen lässt sich ein stabiler Betrieb realisieren. Die OMV-Entwickler haben die GUI bewusst so gestaltet, dass USB für RAID nicht beworben wird — man kann es aber per CLI problemlos umsetzen.

## Warum alte Hardware?

Weil viele 10-15 Jahre alte NAS-Geräte hardwareseitig noch tadellos funktionieren, aber das Hersteller-OS keine Updates mehr bekommt. Mit einem aktuellen Debian + OMV werden diese Geräte wieder voll nutzbar — solange man die Eigenheiten der alten BIOSe und Chipsätze kennt und gegebenenfalls per DSDT-Override umgehen kann.

## Wichtiger Hinweis

**RAID ist kein Backup.** Dieses Setup schützt vor Hardware-Ausfall einzelner Platten, aber nicht vor Ransomware, versehentlichem Löschen, Dateisystem-Korruption, Diebstahl oder Überspannung. Wer Daten wirklich sichern will, braucht zusätzlich ein separates Offline- oder Offsite-Backup.

## Lizenz

MIT — siehe [LICENSE](LICENSE). Nutzung, Anpassung und Weitergabe ausdrücklich erwünscht.

## Autor

**Phydran6** — erstellt im Rahmen konkreter Produktiv-Setups.

Die Anleitungen wurden in Zusammenarbeit mit **Claude Opus 4.7** (Anthropic) erarbeitet und dokumentiert — jeder einzelne Schritt wurde im laufenden Betrieb getestet, bevor er hier festgehalten wurde.

## Mitwirken

Verbesserungsvorschläge, Korrekturen und Erweiterungen sind willkommen — gerne via Issue oder Pull Request.
