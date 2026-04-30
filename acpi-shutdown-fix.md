# ACPI Shutdown Fix per DSDT-Override

Anleitung zum Reparieren von ACPI-BIOS-Bugs auf alten Mainboards, die zu hängenden Shutdowns oder anderen Fehlfunktionen führen — durch Patchen der DSDT-Tabelle und Einbinden über das Initramfs.

Diese Anleitung dokumentiert exemplarisch den Fix für eine **Wortmann TERRA NASBOX 5 G2** (= Thecus N5550, Atom D2550) unter **OMV 7 / Debian 12 / Kernel 6.1**, bei der zwei BIOS-ACPI-Bugs den sauberen Shutdown verhinderten.

## TL;DR

1. ACPI-Tabellen aus dem laufenden System dumpen
2. DSDT mit `iasl` zu lesbarem ASL-Code dekompilieren
3. Fehlerhafte Verweise auskommentieren / fehlerhafte Werte korrigieren
4. OEM-Revision erhöhen (sonst übernimmt der Kernel den Patch nicht)
5. Neu kompilieren zu `dsdt.aml`
6. Als Initramfs-Image mit korrektem Pfad (`kernel/firmware/acpi/dsdt.aml`) ablegen
7. GRUB anweisen, das Image **vor** der eigentlichen Initrd zu laden

## Wann ist diese Anleitung relevant?

Wenn dein System eines oder mehrere der folgenden Symptome zeigt:

- Shutdown hängt mit Meldungen wie
  ```
  ACPI BIOS Error (bug): Could not resolve symbol [\_SB.PCI0.LPCB.TPM.PTS], AE_NOT_FOUND
  ACPI Error: Aborting method \_PTS due to previous error (AE_NOT_FOUND)
  reboot: Power down
  ```
- Hardware geht nach „reboot: Power down" nicht aus (Lüfter und LEDs bleiben an)
- BIOS bietet kein Update mehr an oder Hersteller ist EOL/insolvent
- TPM ist im BIOS nicht abschaltbar oder gar nicht physisch vorhanden trotz ACPI-Verweisen

Reboot funktioniert in solchen Fällen meist trotzdem, weil dabei ein anderer ACPI-Codepfad (`_RST` statt `_PTS` + `_S5`) durchlaufen wird.

## Hintergrund: Was ist DSDT?

Beim Boot übergibt das BIOS dem Linux-Kernel eine sogenannte **DSDT** (Differentiated System Description Table). Darin steht in einer speziellen Sprache (ASL/AML), wie die ACPI-Methoden des Boards funktionieren — also unter anderem, was beim Shutdown, Sleep, Reboot oder Tastendruck passieren soll.

Wenn das BIOS Bugs in dieser Tabelle hat (Verweise ins Leere, falsche Werte, ungültige Methoden), kann der Kernel beim Aufruf dieser Methoden ins Stocken geraten.

Linux unterstützt seit langer Zeit das **DSDT-Override**: Statt die vom BIOS gelieferte Tabelle zu nutzen, lädt der Kernel beim Boot eine gepatchte Version aus dem Initramfs. Voraussetzung ist, dass der Kernel mit `CONFIG_ACPI_TABLE_UPGRADE=y` gebaut wurde (Standard bei Debian).

## Voraussetzungen

- Debian 12 (oder vergleichbar) mit funktionierendem Boot
- Root-Zugriff (per SSH oder lokal)
- Etwas Geduld und Mut — bei sauberer Arbeit ist das Risiko gering, aber nichts für „mal eben"

> **Sicherheit:** Falls der Patch fehlschlägt, lädt der Kernel wieder die Original-DSDT aus dem BIOS. Ein „Brick" ist mit DSDT-Override nicht möglich. Im Notfall einfach `acpi_override.img` aus `/boot/` löschen und `update-grub` erneut ausführen.

## Schritt-für-Schritt-Anleitung

### 1. Tools installieren

```bash
sudo apt update
sudo apt install acpica-tools -y
```

### 2. Arbeitsverzeichnis und ACPI-Dump

```bash
mkdir ~/dsdt && cd ~/dsdt
sudo acpidump -b
ls
```

Erwartete Ausgabe (je nach Board variabel):
```
apic.dat  facp.dat  hpet.dat  ssdt1.dat  uefi1.dat
dsdt.dat  facs.dat  mcfg.dat  ssdt2.dat  uefi2.dat
...
```

### 3. DSDT (mitsamt SSDTs) dekompilieren

```bash
iasl -e ssdt*.dat -d dsdt.dat
```

Das `-e ssdt*.dat` ist wichtig: Es teilt dem Disassembler die SSDTs als Referenz mit, sodass externe Methodenaufrufe sauber aufgelöst werden können.

Ergebnis: `dsdt.dsl` im Arbeitsverzeichnis. Warnungen (`There were N external control methods found...`) können ignoriert werden.

### 4. Fehlerhafte Stellen finden

Beispiel TPM-Bug:

```bash
grep -n "TPM\|_PTS" dsdt.dsl
```

Typische problematische Zeile:
```
4899:            \_SB.PCI0.LPCB.TPM.PTS (Arg0)
```

Zur Kontrolle die umgebenden Zeilen anschauen:

```bash
sed -n '4895,4905p' dsdt.dsl
```

### 5. Patch 1: TPM-Aufruf in `_PTS` deaktivieren

Datei öffnen:

```bash
nano dsdt.dsl
```

Zur betroffenen Zeile springen (`Strg+_`, dann Zeilennummer eingeben). Den fehlerhaften Aufruf auskommentieren:

```
// \_SB.PCI0.LPCB.TPM.PTS (Arg0)
```

> ASL-Kommentare sind C-Style: `//` für Einzeiler, `/* ... */` für Blöcke.

### 6. Patch 2: `_S5`-State korrigieren

Auf manchen Boards passt der vom BIOS deklarierte S5-State nicht zur tatsächlichen Hardware-Erwartung. Standard-Wert ist `0x07`, manche Chipsätze verlangen `0x05`:

```bash
grep -n "_S5" dsdt.dsl
```

Block ansehen:

```bash
sed -n '5596,5605p' dsdt.dsl
```

Beispiel:
```
Name (_S5, Package (0x04)
{
    0x07,
    Zero,
    Zero,
    Zero
})
```

Die `0x07` zu `0x05` ändern — dies ist hardwarespezifisch und kann ohne Risiko probiert werden, da bei falschem Wert nur der Shutdown-Pfad scheitert (nicht der Boot).

### 7. OEM-Revision erhöhen (PFLICHT)

Damit der Kernel den Patch übernimmt, muss die **OEM Revision höher** sein als die des BIOS-Originals.

Header der `dsdt.dsl` zeigt die aktuelle Revision:
```
* OEM Revision     0x00000001 (1)
```

In der `DefinitionBlock`-Zeile (am Anfang nach dem Header-Kommentar) die Zahl erhöhen:

```
DefinitionBlock ("", "DSDT", 2, "PTL   ", "OEMTABLE", 0x00000002)
```

Bei jedem weiteren Patch-Durchlauf: weiter erhöhen (`0x00000003`, `0x00000004` etc.).

> Ohne diesen Schritt findet der Kernel das Override zwar im Initrd, übernimmt es aber nicht — erkennbar daran, dass `cat /sys/firmware/acpi/tables/DSDT | grep -c "TPM.PTS"` weiterhin `1` (oder mehr) zurückgibt.

### 8. Neu kompilieren

```bash
iasl -tc dsdt.dsl
```

Wichtig: Das Ergebnis muss `Compilation successful. 0 Errors` zeigen. Warnings und Remarks sind harmlos (waren schon im BIOS-Original so).

Ergebnis: `dsdt.aml` im Arbeitsverzeichnis.

### 9. Initramfs-Image bauen

Der Kernel erwartet das Override-Image in einer **exakten Verzeichnisstruktur**: `kernel/firmware/acpi/dsdt.aml`.

```bash
sudo mkdir -p /boot/acpi_override/kernel/firmware/acpi
sudo cp dsdt.aml /boot/acpi_override/kernel/firmware/acpi/
cd /boot/acpi_override
sudo find kernel | sudo cpio -H newc --create > /boot/acpi_override.img
```

Erwartete Ausgabe: `XX Blöcke` (Größe variiert).

### 10. GRUB konfigurieren

```bash
sudo nano /etc/default/grub
```

Über die `GRUB_DEFAULT`-Zeile einfügen:

```
GRUB_EARLY_INITRD_LINUX_CUSTOM="acpi_override.img"
```

> Kein führender Slash — GRUB erwartet hier den Namen relativ zum Boot-Verzeichnis.

Speichern und GRUB aktualisieren:

```bash
sudo update-grub
```

Kontrolle der generierten GRUB-Config:

```bash
grep "initrd" /boot/grub/grub.cfg | head -3
```

Erwartete Ausgabe (verkürzt):
```
initrd  /boot/acpi_override.img /boot/initrd.img-6.1.0-XX-amd64
```

Das `acpi_override.img` muss **vor** der eigentlichen Initrd erscheinen.

### 11. Reboot und Verifizieren

```bash
sudo reboot
```

Nach dem Reboot prüfen, ob der Patch aktiv ist:

```bash
sudo cat /sys/firmware/acpi/tables/DSDT | grep -c "TPM.PTS"
```

- `0` → Patch ist aktiv ✅
- `>0` → Patch wurde nicht übernommen (häufigste Ursache: OEM-Revision wurde nicht erhöht)

Zusätzlich kann man im dmesg nach Override-Hinweisen suchen:

```bash
sudo dmesg | grep -i "dsdt\|acpi.*override"
```

### 12. Shutdown-Test

```bash
sudo poweroff
```

Hardware sollte jetzt sauber ausgehen (LEDs aus, Lüfter aus).

## Troubleshooting

### Patch wird nicht übernommen

1. **OEM-Revision erhöht?** Wichtigster Punkt. Bei jedem Patch-Durchlauf weiter erhöhen.
2. **Pfad korrekt?** Image muss `kernel/firmware/acpi/dsdt.aml` enthalten.
3. **Reihenfolge in GRUB?** `acpi_override.img` muss **vor** der eigentlichen Initrd geladen werden.
4. **Kernel-Config aktiv?** Prüfen mit `zgrep "ACPI_TABLE_UPGRADE" /boot/config-$(uname -r)`. Sollte `=y` sein.

### Kompilierung schlägt fehl

`iasl -tc` zeigt Errors statt nur Warnings. Mögliche Ursachen:
- Tippfehler beim Patchen → Diff zur Original-DSL anschauen
- Klammerstruktur kaputt → `If`/`Else`-Blöcke prüfen
- Im Notfall: `dsdt.dsl` neu aus `dsdt.dat` erzeugen und Patch neu durchführen

### Boot bleibt hängen nach Override

Sehr selten — würde bedeuten, dass der gepatchte Code Methoden korrumpiert hat, die beim Boot benötigt werden. Notfall-Recovery:

1. Im GRUB-Menü beim Booten `e` drücken
2. Die `initrd`-Zeile editieren: `/boot/acpi_override.img` herauslöschen
3. Mit `Ctrl+X` einmalig booten
4. Im laufenden System `sudo rm /boot/acpi_override.img && sudo update-grub`

## Was tun, wenn ein Patch nicht funktioniert?

Nicht jeder ACPI-Bug ist patchbar. Manche Probleme liegen im Chipsatz selbst und werden nur über Code in der DSDT *ausgelöst*, aber nicht *verursacht*.

Im Beispiel dieser Anleitung war der TPM-Bug per Auskommentieren behebbar (kosmetisch + verhindert den Method-Abort), aber der eigentliche S5-Hänger erforderte zusätzlich die Korrektur des `_S5`-Werts.

Wenn Patches nichts bringen, sind realistische Alternativen:
- **Smart-Plug** (z. B. Shelly) vor der Steckdose, die nach einem Shutdown-Skript automatisch trennt
- **Watchdog-Timer**, der die Box nach festgelegter Zeit hart zurücksetzt
- **Damit leben** — Reboot funktioniert in den meisten Fällen, Shutdown wird im NAS-Alltag selten gebraucht

## Auswirkungen auf Kernel-Updates

Der Override ist robust gegen normale Kernel-Updates:

- `acpi_override.img` liegt in `/boot/` und wird nicht von Kernel-Paketen angefasst
- `GRUB_EARLY_INITRD_LINUX_CUSTOM` in `/etc/default/grub` bleibt erhalten
- `update-grub` läuft bei Kernel-Updates automatisch und integriert das Override in die neuen Boot-Einträge

Nach Kernel-Updates ist also kein erneutes Patchen nötig. Schnelltest nach Updates:

```bash
sudo cat /sys/firmware/acpi/tables/DSDT | grep -c "TPM.PTS"
```

Sollte weiterhin `0` zurückgeben.

Bei einem **Major-Distribution-Upgrade** (z. B. Debian 12 → 13) kann es vorkommen, dass die GRUB-Config neu generiert wird und der Override-Eintrag verschwindet. In diesem Fall einfach den `GRUB_EARLY_INITRD_LINUX_CUSTOM`-Eintrag wieder ergänzen und `update-grub` ausführen.

## Backup der gepatchten Dateien

Es lohnt sich, die fertigen Patch-Dateien zu sichern — entweder lokal oder in einem privaten Repo:

```bash
cd ~
tar czf dsdt-backup-$(date +%Y%m%d).tar.gz dsdt/dsdt.dsl dsdt/dsdt.aml /boot/acpi_override.img
```

Bei einer Neuinstallation des Systems lassen sich die Patches so schnell wieder einspielen, ohne den ganzen Disassembly-/Patch-Prozess zu wiederholen.

## Weiterführendes

- [Linux Kernel: ACPI Custom Tables](https://www.kernel.org/doc/Documentation/acpi/initrd_table_override.txt)
- [Intel ACPI Component Architecture (ACPICA)](https://acpica.org/) — Hersteller von `iasl`/`acpidump`
- [ASL/AML Spezifikation](https://uefi.org/specifications) — für tieferen Einstieg in ACPI-Patching
