# Five - HackMyVM (Medium) - Writeup

![Five Icon](Five.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Five" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Five (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Five](https://hackmyvm.eu/machines/machine.php?vm=Five)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 8. Oktober 2022
*   **Original Writeup:** [https://alientec1908.github.io/Five_HackMyVM_Medium/](https://alientec1908.github.io/Five_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mit `nmap` offenbarte lediglich einen offenen Port 80, auf dem ein Nginx-Webserver lief und einen `403 Forbidden`-Fehler zurückgab. Die `robots.txt` verwies jedoch auf ein `/admin`-Verzeichnis. Mittels `gobuster` wurden trotz des 403-Fehlers weitere Pfade wie `/uploads`, `/upload.html` und `/upload.php` entdeckt.

Über die Upload-Funktionalität (`/upload.html`, `/upload.php`) wurde eine vorbereitete PHP-Reverse-Shell (`rev.php`) hochgeladen. Der Aufruf dieser Shell (vermutlich unter `/uploads/rev.php`) via `curl` stellte eine Verbindung zu einem `nc`-Listener her und gewährte initialen Zugriff als Benutzer `www-data`. Die Shell wurde anschließend stabilisiert (`python pty`).

Die Analyse der `sudo`-Rechte (`sudo -l`) für `www-data` zeigte, dass der Befehl `/bin/cp` als Benutzer `melisa` ohne Passwort ausgeführt werden durfte. Dies wurde ausgenutzt, um SSH-Zugriff als `melisa` zu erlangen: Entweder durch Kopieren ihres privaten Schlüssels (`/home/melisa/.ssh/id_rsa`) nach `/tmp` oder durch Überschreiben ihrer `/home/melisa/.ssh/authorized_keys` mit einem eigenen öffentlichen Schlüssel mittels `sudo -u melisa cp`. Eine Prüfung mit `ss` zeigte einen SSH-Dienst auf Port `4444` (localhost). Der Login als `melisa` erfolgte via `ssh -i <schlüssel> melisa@localhost -p 4444`.

Eine erneute Prüfung der `sudo`-Rechte als `melisa` offenbarte die Möglichkeit, `/bin/man` als `root` ohne Passwort auszuführen. Durch Ausführen von `sudo /bin/man man` und anschließender Eingabe von `!/bin/sh` im `man`-Pager (less) wurde eine Root-Shell erlangt. Schließlich wurden die User- und Root-Flags ausgelesen.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `vi`
*   `gobuster`
*   `grep`
*   `ip`
*   `msfconsole` (multi/handler - erfolglos)
*   `nc` (netcat)
*   `curl`
*   `python3` (`pty` module)
*   `sudo`
*   `touch`
*   `cp`
*   `chmod`
*   `ssh-keygen` (potenziell)
*   `ss`
*   `ssh`
*   `ls`
*   `man`
*   `cat`

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Portscan (`nmap`) -> Port 80 (Nginx, 403 Forbidden), `/admin` in `robots.txt`.
2.  **Web Enumeration:** Verzeichnissuche (`gobuster`) -> findet `/uploads`, `/upload.html`, `/upload.php`.
3.  **Vorbereitung:** PHP-Reverse-Shell (`rev.php`) mit korrekter IP/Port konfigurieren (`grep`, `ip a`).
4.  **Initial Access:** `rev.php` über `/upload.html` hochladen. `nc`-Listener starten. Shell via `curl http://<IP>/uploads/rev.php` auslösen -> Shell als `www-data`.
5.  **Shell Stabilization:** `python3 -c 'import pty; ...'`, `export TERM=xterm`.
6.  **Privilege Escalation (www-data -> melisa):** `sudo -l` zeigt `(melisa) NOPASSWD: /bin/cp`. Eigenen Public-Key (`authorized_keys`) nach `/tmp` schreiben. Mit `sudo -u melisa cp /tmp/authorized_keys /home/melisa/.ssh/authorized_keys` überschreiben (oder privaten Schlüssel kopieren). `ss` zeigt SSH auf `localhost:4444`. Login mit `ssh -i <key> melisa@localhost -p 4444`.
7.  **Privilege Escalation (melisa -> root):** `sudo -l` zeigt `(ALL) NOPASSWD: /bin/man`. Ausführen mit `sudo /bin/man man`. Im Pager `!/bin/sh` eingeben -> Root-Shell.
8.  **Flags:** `cat /home/melisa/user.txt` und `cat /root/root.txt` auslesen.

## Flags

*   **User Flag (`/home/melisa/user.txt`):** `Ilovebinaries`
*   **Root Flag (`/root/root.txt`):** `WTFGivemefive`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
