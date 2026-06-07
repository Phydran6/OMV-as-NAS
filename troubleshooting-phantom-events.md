# Tägliche „DeviceDisappeared / NewArray"-Phantom-Events (mdadm)

Nach dem Upgrade auf **OMV 8** kommt täglich eine Cron-Mail von
`/etc/cron.daily/openmediavault-mdadm`, obwohl das Array kerngesund ist. Diese
Anleitung erklärt die eigentliche Ursache und behebt sie dauerhaft mit einer
lokalen udev-Regel — ohne Reboot und ohne Array-Downtime.

## Symptom

Täglich landet eine `cron.daily`-Mail mit folgendem Inhalt im Postfach:

```
/etc/cron.daily/openmediavault-mdadm:
mdadm: DeviceDisappeared event detected on md device /dev/md/md0
mdadm: NewArray event detected on md device /dev/md0
```

Trotz der alarmierenden Wortwahl ist das Array gesund (`cat /proc/mdstat` zeigt
`[UUUU]`). Es handelt sich um reine Phantom-Events — kosmetisch, ohne realen
Defekt.

## Ursache

Dies ist ein OMV-8-Problem (siehe
[openmediavault/openmediavault#2071](https://github.com/openmediavault/openmediavault/issues/2071)):
Der by-name-Symlink `/dev/md/md0` fehlt.

Der Pfadunterschied in den beiden Meldungen ist der Hinweis: `/dev/md/md0`
(by-name-Symlink) vs. `/dev/md0` (Device-Node). Der Cronjob ruft
`mdadm --monitor --scan --oneshot` auf, das die Arrays unter `/dev/md/` sucht.
Fehlt der Symlink, meldet mdadm den Pfad als verschwunden (`DeviceDisappeared`)
und den rohen Device-Node `/dev/md0` als neu (`NewArray`) — bei jedem Lauf aufs
Neue.

Debian liefert die Regel, die den Symlink eigentlich anlegen sollte, bereits mit
(`/usr/lib/udev/rules.d/63-md-raid-arrays.rules`):

```
ENV{DEVTYPE}=="disk", ENV{MD_DEVNAME}=="?*", SYMLINK+="md/$env{MD_DEVNAME}"
```

Sie greift aber nur, wenn `MD_DEVNAME` gesetzt ist. Bei Arrays, deren Name einen
Host-Präfix trägt (`<hostname>:0`) und nicht als lokaler Host aufgelöst wird,
exportiert neueres mdadm `MD_DEVNAME` **nicht** — also wird kein Symlink
angelegt.

Ursache bestätigen:

```bash
ls -la /dev/md/                                 # leer oder „No such file or directory"
mdadm --detail --no-devices --export /dev/md0
```

Erwarteter Export — `MD_NAME` vorhanden, `MD_DEVNAME` fehlt:

```
MD_LEVEL=raid6
MD_DEVICES=4
MD_METADATA=1.2
MD_UUID=<array-uuid>
MD_NAME=<hostname>:0
```

## Fix

Eine lokale udev-Regel ergänzen, die den Symlink aus dem Kernel-Namen (`%k`)
anlegt — unabhängig von `MD_DEVNAME`. Dafür eine **eigene** Datei verwenden;
OMVs eigene `99-openmediavault-md-raid.rules` **nicht** editieren, sie wird bei
Updates überschrieben.

```bash
cat > /etc/udev/rules.d/99-local-md-symlink.rules << 'EOF'
SUBSYSTEM=="block", ACTION=="add|change", KERNEL=="md[0-9]*", ENV{DEVTYPE}=="disk", SYMLINK+="md/%k"
EOF

udevadm control --reload-rules
udevadm trigger --subsystem-match=block
```

## Verifizieren

```bash
ls -la /dev/md/                         # md0 -> ../md0
mdadm --monitor --scan --oneshot        # keine Ausgabe = keine Events
```

Optional den Cronjob manuell triggern und die Logs prüfen:

```bash
run-parts /etc/cron.daily/
journalctl --since "2 minutes ago" | grep -iE "DeviceDisappeared|NewArray"
```

Erwartet: keine Treffer mehr. Damit hören auch die täglichen Mails auf.

## Hinweise

- **Reboot-fest:** udev legt den Symlink für jedes `md`-Device neu an, das
  hochkommt.
- **`%k` folgt dem Kernel-Namen automatisch.** Startet das Array nach einem
  Kernel-Update als `/dev/md127`, wandert der Symlink mit. OMV passt
  `mdadm.conf` beim nächsten Apply entsprechend an.
- **Das ist ein Workaround für einen Upstream-OMV-Bug, kein Root-Cause-Fix.**
  Die saubere Alternative — das Array unter seinem by-name-Pfad neu zusammenbauen
  (`/dev/md/0`), sodass die Debian-Standardregel den Symlink erzeugt — erfordert
  Array-Downtime (und ein POSIX-konformes Umbenennen, da ein Doppelpunkt in
  `<hostname>:0` als Dateiname nicht erlaubt ist). Für ein kosmetisches Problem
  lohnt das nicht.
- **`command not found` trotz vorhandenem Binary:** Wird `mdadm` oder
  `update-initramfs` als fehlend gemeldet, obwohl `/usr/sbin/…` existiert, ist
  `/usr/sbin` nicht im `$PATH`. Tritt typischerweise auf, wenn man mit `su`
  (statt `su -`) zu root wechselt — das übernimmt den User-PATH. Lösung: mit
  `su -` oder `sudo -i` einloggen, oder den vollen Pfad nutzen.

## Verwandte Dokumente

- [USB-RAID Setup](setup.md) — Aufbau des RAID 6 und Persistieren der
  `mdadm.conf`
