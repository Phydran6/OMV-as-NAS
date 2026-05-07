### Tägliche „DeviceDisappeared / NewArray"-Mails

[#tägliche-devicedisappeared--newarray-mails](#t%C3%A4gliche-devicedisappeared--newarray-mails)

OMV verschickt täglich eine `cron.daily`-Mail mit folgendem Inhalt:

```
/etc/cron.daily/openmediavault-mdadm:
mdadm: DeviceDisappeared event detected on md device /dev/md/md0
mdadm: NewArray event detected on md device /dev/md0
```

Trotz der alarmierenden Wortwahl ist das Array gesund (`cat /proc/mdstat` zeigt `[UUUU]`). Es handelt sich um Phantom-Events.

#### Ursache

[#ursache](#ursache)

Der Pfadunterschied in den beiden Meldungen ist der Hinweis: `/dev/md/md0` (by-name-Symlink) vs. `/dev/md0` (Device-Node). Beim Aufruf von `mdadm --monitor --scan --oneshot` (was der OMV-Cronjob macht) sucht mdadm den by-name-Symlink unter `/dev/md/`, findet ihn nicht → meldet `DeviceDisappeared`. Dann sieht es `/dev/md0` → meldet `NewArray`. Beim nächsten Lauf das gleiche Spiel.

Der Symlink fehlt, weil die `ARRAY`-Zeile in `/etc/mdadm/mdadm.conf` als Device-Pfad direkt `/dev/md0` referenziert statt eines by-name-Pfades wie `/dev/md/<name>`.

Prüfen:

```bash
ls -la /dev/md/
```

Wenn das Verzeichnis nicht existiert oder den erwarteten Symlink nicht enthält, liegt genau dieses Problem vor.

#### Stolperfalle: POSIX-Kompatibilität des Array-Namens

[#stolperfalle-posix-kompatibilität-des-array-namens](#stolperfalle-posix-kompatibilit%C3%A4t-des-array-namens)

Der Default-Array-Name nach `mdadm --create` enthält oft einen Doppelpunkt, z. B. `<hostname>:0`. Ein Doppelpunkt ist in POSIX-Dateinamen nicht erlaubt — mdadm verwirft den Pfad mit:

```
mdadm: Value "<hostname>:0" cannot be set as devname. Reason: Not POSIX compatible. Value ignored.
```

Daher muss der by-name-Pfad in der Config einen POSIX-konformen Namen verwenden (z. B. `<hostname>_0` mit Unterstrich statt Doppelpunkt).

#### Fix

[#fix](#fix)

Backup, dann ARRAY-Zeile auf einen by-name-Pfad mit POSIX-konformem Namen umschreiben:

```bash
cp /etc/mdadm/mdadm.conf /etc/mdadm/mdadm.conf.bak
sed -i 's|ARRAY /dev/md0 |ARRAY /dev/md/<hostname>_0 |' /etc/mdadm/mdadm.conf
update-initramfs -u
reboot
```

Nach dem Reboot verifizieren:

```bash
ls -la /dev/md/
```

Erwartet: ein Symlink `<hostname>_0 -> ../md0`.

Den Cronjob manuell triggern und Logs prüfen:

```bash
run-parts /etc/cron.daily/
journalctl --since "2 minutes ago" | grep -iE "DeviceDisappeared|NewArray"
```

Erwartet: keine Treffer mehr. Damit hören auch die täglichen Mails auf.

#### Hinweis: `update-initramfs: command not found`

[#hinweis-update-initramfs-command-not-found](#hinweis-update-initramfs-command-not-found)

Wenn `update-initramfs` als angeblich fehlend gemeldet wird, obwohl `/usr/sbin/update-initramfs` existiert, ist `/usr/sbin` nicht im `$PATH`. Tritt typischerweise auf, wenn man mit `su` (statt `su -`) zu root wechselt — das übernimmt den User-PATH. Lösung: mit `su -` oder `sudo -i` einloggen, oder den vollen Pfad nutzen:

```bash
/usr/sbin/update-initramfs -u
```
