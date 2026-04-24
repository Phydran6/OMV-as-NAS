# OMV USB-RAID 6 mit Hotspares

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![OpenMediaVault](https://img.shields.io/badge/OpenMediaVault-7%2F8-blue)](https://www.openmediavault.org/)
[![Platform: Debian](https://img.shields.io/badge/platform-Debian-red)](https://www.debian.org/)

Eine komplette, praxiserprobte Anleitung zum Aufsetzen eines Software-RAID 6 mit mehreren Hotspares auf USB-angeschlossenen Festplatten unter OpenMediaVault (OMV) — inklusive Lösung für die OMV-Einschränkung, dass USB-Platten in der Web-UI beim RAID-Erstellen und beim Hinzufügen von Spares ausgeblendet werden.

## Was dieses Repo abdeckt

- Software-RAID 6 mit mehreren Hotspares per `mdadm` (CLI)
- Strategische Rollenverteilung der Platten nach SMART-Werten
- Dauerhafte Konfiguration inklusive Initramfs-Update
- E-Mail-Benachrichtigungen bei Plattenausfällen
- Failover-Tests (Software und Hardware)
- Dauerhafter Custom-`From`-Header für Notification-Mails via Salt-State (überlebt OMV-Updates)

## Schnellnavigation

- [Vollständige Anleitung](docs/setup.md)
- [Mail-Absender anpassen](docs/setup.md#mail-absender-dauerhaft-ändern-salt-state)
- [Lizenz](./LICENSE)

## Zielgruppe

Admins und Homelab-Enthusiasten, die:

- ein OMV-NAS mit USB-Festplatten betreiben (oder planen)
- ein RAID mit maximaler Ausfallsicherheit wollen
- nicht auf die OMV-GUI beschränkt sein möchten, wenn diese USB-Geräte blockiert
- ihre Konfiguration dauerhaft und update-sicher gestalten wollen

## Warum USB + RAID?

Kurz: Weil es geht und es für viele Setups die günstigste Lösung ist. Lang: USB-Bridges gelten zwar als unzuverlässiger als direkte SATA/SAS-Anbindung (Spin-Down, Reconnect-Probleme, gelegentlich vertauschte Serials), aber mit der richtigen RAID-Konfiguration (RAID 6 + Hotspares) und angemessenen Benachrichtigungen lässt sich ein stabiler Betrieb realisieren. Die OMV-Entwickler haben die GUI bewusst so gestaltet, dass USB für RAID nicht beworben wird — man kann es aber per CLI problemlos umsetzen.

## Wichtiger Hinweis

**RAID ist kein Backup.** Dieses Setup schützt vor Hardware-Ausfall einzelner Platten, aber nicht vor Ransomware, versehentlichem Löschen, Dateisystem-Korruption, Diebstahl oder Überspannung. Wer Daten wirklich sichern will, braucht zusätzlich ein separates Offline- oder Offsite-Backup.

## Lizenz

MIT — siehe [LICENSE](./LICENSE). Nutzung, Anpassung und Weitergabe ausdrücklich erwünscht.

## Autor

**Phydran6** — erstellt im Rahmen eines konkreten Produktiv-Setups.

Die Anleitung wurde in Zusammenarbeit mit **Claude Opus 4.7** (Anthropic) erarbeitet und dokumentiert — jeder einzelne Schritt wurde im laufenden Betrieb getestet, bevor er hier festgehalten wurde.

## Mitwirken

Verbesserungsvorschläge, Korrekturen und Erweiterungen sind willkommen — gerne via Issue oder Pull Request.
