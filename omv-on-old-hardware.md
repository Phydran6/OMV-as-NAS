# OMV 7 auf alter NAS-Hardware (Cedarview & Co.)

Anleitung für die Installation von OpenMediaVault 7 auf älteren NAS-Geräten mit Intel-Atom-CPUs der Cedarview-Generation (D2550, D2700 etc.). Beispielgerät dieser Anleitung: **Wortmann TERRA NASBOX 5 G2** (umgelabelte **Thecus N5550**).

## Hardware-Profil

Diese Anleitung richtet sich an Geräte mit folgendem Profil:

| Komponente | Spezifikation |
|---|---|
| CPU | Intel Atom D2550 (Cedarview-D, 2C/4T, 1,86 GHz, 10W TDP, x86_64) |
| RAM | 2 GB DDR3 (auf 4 GB erweiterbar mit 1,5V SO-DIMM) |
| Bays | 5× 3,5"/2,5" SATA Hot-Swap |
| Netzwerk | 2× Gigabit Ethernet |
| Original-OS | ThecusOS Linux (EOL, keine Updates mehr) |
| Boot | BIOS-only, **kein UEFI** |
| Bootmedium | mSATA-SSD (typ. 24 GB), USB-Stick, oder eines der SATA-Bays |

Vergleichbare Geräte mit gleichem Vorgehen: andere Thecus-Modelle der gleichen Generation (N4800/N5550/N7700 mit Cedarview), QNAP TS-x69-Serie, kleine Synology-Modelle der 2012-2014er Jahre.

## Warum nicht TrueNAS?

TrueNAS CORE oder SCALE sind keine sinnvolle Option auf dieser Hardware:
- Beide setzen offiziell mindestens 8 GB RAM voraus
- ZFS, das Standard-Dateisystem, frisst viel RAM und ist mit 2 GB nicht stabil betreibbar

OpenMediaVault mit `mdadm` + `ext4` läuft auf 2 GB RAM problemlos für reines Fileserving (SMB/NFS) — solange keine Container, kein Docker und kein ZFS dazukommen.

## Versionsproblem: Warum OMV 7 statt OMV 8

Cedarview-CPUs mit modernen Linux-Kerneln zu kombinieren, scheitert oft am Kernel:

| OMV-Version | Debian-Basis | Kernel | Cedarview-Kompatibilität |
|---|---|---|---|
| OMV 6 | Debian 11 (Bullseye) | 5.10 LTS | ✅ funktioniert |
| OMV 7 | Debian 12 (Bookworm) | 6.1 LTS | ✅ funktioniert |
| OMV 8 | Debian 13 (Trixie) | 6.12 | ❌ Bootet nicht (hängt nach „Loading initial ramdisk") |

**Empfehlung:** OMV 7 (Debian 12, Kernel 6.1 LTS) — Sicherheitsupdates sind durch die Debian-LTS-Pflege bis Mitte 2028 garantiert.

OMV 8 hängt bei dieser Hardware bereits beim Kernel-Boot. Der `i915`-Treiber für die GMA 3650 (PowerVR-basiert) wurde in neueren Kerneln entfernt, dazu kommen ACPI-Probleme bei Cedarview. `nomodeset` als Boot-Parameter hilft nicht, weil das Problem tiefer liegt.

OMV 8 lässt sich nicht auf Debian 12 installieren — das Install-Skript prüft die Debian-Version und verweigert die Installation.

## Voraussetzungen

- Bootmedium für OMV (typ. die mSATA-SSD)
- USB-Stick (≥ 2 GB) für den Installer
- Tastatur und Monitor für die initiale Installation
- Netzwerkanschluss
- Optional: 4-GB-DDR3-SO-DIMM (1,5 V) für RAM-Upgrade — empfehlenswert wenn später Container/Plugins geplant sind

## Installation

### 1. ISO herunterladen

OMV 7 ISO von SourceForge:

```
https://sourceforge.net/projects/openmediavault/files/
```

Pfad: `iso/` → `7.x.x/` → `openmediavault_7.4.17-amd64.iso` (ca. 700 MB).

7.4.17 ist mit hoher Wahrscheinlichkeit die letzte oder fast letzte Major-Version der 7er-Reihe — Bugfixes erscheinen weiter, aber keine neuen Features. Updates auf der OMV-7-Linie kommen automatisch über das Webinterface.

### 2. USB-Stick beschreiben

- **Windows:** [Rufus](https://rufus.ie) — Standardeinstellungen, „im DD-Modus schreiben" wenn gefragt
- **Linux/Mac:** [Balena Etcher](https://www.balena.io/etcher/) oder `dd`

### 3. Vom Stick booten

USB-Stick einstecken, NAS einschalten, Boot-Menü-Hotkey drücken (oft `F11` oder `F12`, je nach Board). USB-Stick auswählen.

### 4. Im OMV-Installer

| Einstellung | Wert |
|---|---|
| Sprache | Deutsch (oder nach Wahl) |
| Hostname | nach Wahl, z. B. `nas` |
| Domain | leer lassen |
| Root-Passwort | sicher setzen |
| Festplatte zur Installation | **mSATA-SSD** (typ. 24 GB) — **NICHT** den USB-Stick und **NICHT** versehentlich eine Datenplatte |

> **Tipp:** Datenplatten vor der Installation aus den Bays nehmen, damit man sie im Installer-Menü gar nicht erst auswählen kann.

### 5. Nach Reboot

USB-Stick rausziehen. System sollte von der mSATA-SSD booten. Webinterface erreichbar unter:

```
http://<NAS-IP>
```

Default-Login: `admin` / `openmediavault`

## Häufige Probleme nach der Installation

### Boot bleibt nach „Loading initial ramdisk" hängen
Du hast wahrscheinlich versehentlich OMV 8 installiert. Lösung: OMV 7 ISO nehmen.

### Boot funktioniert, aber Shutdown hängt
ACPI-BIOS-Bug. Siehe separates Dokument: **[ACPI Shutdown Fix](acpi-shutdown-fix.md)**.

### Webinterface zeigt „Software Failure" oder ist nicht erreichbar
- Boot ggf. noch nicht durch — 1-2 Minuten warten
- Netzwerk-Setup im Installer prüfen (DHCP vs. statisch)
- Login per SSH und `systemctl status openmediavault-engined nginx-openmediavault`

### Display und Front-Buttons der NAS-Hardware tot
Erwartet. LCD und Tasten der Front sind proprietär an das Original-OS gekoppelt und werden unter OMV nicht mehr funktionieren. Headless-Betrieb ist der vorgesehene Modus.

## Lebensdauer und Support-Strategie

- **OMV 7 + Debian 12** läuft mit Sicherheitsupdates bis **Juni 2028** (Debian LTS)
- Innerhalb der OMV-7-Reihe können Minor-Updates (7.x → 7.y) problemlos eingespielt werden
- Wenn OMV 9 / Debian 14 erscheint: bei OMV 7 bleiben, kein Upgrade
- Hardware ist 2013 — irgendwann werden Festplatten oder Netzteile sterben, plane das ein
- Realistische Strategie: bis 2028 nichts ändern, dann Hardware-Wechsel oder Neukauf

## Verwandte Dokumente

- [ACPI Shutdown Fix (DSDT-Override)](acpi-shutdown-fix.md) — Pflichtlektüre, falls Shutdown hängt
- [USB-RAID Setup](setup.md) — wenn die Datenplatten extern via USB hängen sollen
