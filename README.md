# Oldschool - HackMyVM (Medium)

![Oldschool.png](Oldschool.png)

## Übersicht

*   **VM:** Oldschool
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Oldschool)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 11. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Oldschool_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Oldschool" zu erlangen. Der initiale Zugriff erfolgte durch Ausnutzung einer Kombination aus einer unsicheren Zugriffskontrolle auf eine Webanwendung (`verification.php` konnte durch Setzen des `Role: admin`-Headers umgangen werden) und einer Command Injection-Schwachstelle im `pingertool2000.php`. Dies ermöglichte eine Reverse Shell als `www-data`. Nach der Identifizierung eines TFTP-Dienstes und dem Herunterladen einer Konfigurationsdatei (`passwd.cfg`) wurde ein LessPass-Seed für den Benutzer `fanny` gefunden. Mit dem generierten Passwort wurde Telnet-Zugriff als `fanny` erlangt. Die finale Rechteausweitung zu Root gelang durch Ausnutzung einer unsicheren `sudo`-Regel, die `fanny` erlaubte, `/usr/bin/nano /etc/snmp/snmpd.conf` als Root ohne Passwort auszuführen, was die Ausführung von Shell-Befehlen innerhalb von `nano` ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   Burp Suite (impliziert für Header-Manipulation und POST-Requests)
*   `nc` (netcat)
*   `ls`
*   `id`
*   `uname`
*   `cd`
*   `find`
*   `tftp` client
*   `msfconsole` (Metasploit, für TFTP-Brute-Force)
*   `cat`
*   LessPass (impliziert/extern zur Passwortgenerierung)
*   `telnet`
*   `sudo`
*   `nano`
*   Standard Linux-Befehle (`echo`, `bash`, `sh`, `stty`, `fg`, `export`, `mv`, `pwd`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Oldschool" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.126) mit `arp-scan` identifiziert. Hostname `oldschool.hmv` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 23 (Telnet) und Port 80 (HTTP, Apache 2.4.54).
    *   `gobuster` auf Port 80 fand u.a. `index.php`, `verification.php` und `denied.php`.

2.  **Initial Access (Header Bypass & RCE als `www-data`):**
    *   Ein GET-Request an `/verification.php` wurde mit Burp Suite abgefangen. Durch Hinzufügen des HTTP-Headers `Role: admin` wurde die Zugriffskontrolle umgangen, was zu einer Weiterleitung auf `pingertool2000.php` führte.
    *   Die Seite `pingertool2000.php` war anfällig für Command Injection im POST-Parameter `url`. Eine Filterung (Verbot von "php") wurde durch Base64-Kodierung des Payloads und Shell-Piping umgangen: `hackmyvm.eu${IFS}%0aec''ho%09"BASE64_REVERSE_SHELL"%09|%09base64%09-d%09|%09bas''h`.
    *   Eine Reverse Shell als `www-data` wurde auf einem Netcat-Listener empfangen.

3.  **Privilege Escalation (von `www-data` zu `fanny` via TFTP & LessPass):**
    *   Als `www-data` wurde ein TFTP-Dienst auf dem Ziel identifiziert (Port 69/udp).
    *   Mittels Metasploit (`auxiliary/scanner/tftp/tftpbrute`) wurde die Datei `passwd.cfg` auf dem TFTP-Server gefunden.
    *   `passwd.cfg` enthielt einen LessPass-Seed (`14mw0nd32fu1`) für `oldschool.hmv` und den Benutzer `fanny`.
    *   Mit den LessPass-Informationen wurde das Passwort `44Tg".P0/jKo_'t:` für `fanny` generiert.
    *   Erfolgreicher Telnet-Login als `fanny` mit dem generierten Passwort.
    *   Die User-Flag (`a8a1d4b692341fc2c59ca33815fb59d6`) wurde in `/home/fanny/user.txt` gefunden.

4.  **Privilege Escalation (von `fanny` zu `root` via `sudo nano`):**
    *   `sudo -l` als `fanny` zeigte, dass der Befehl `/usr/bin/nano /etc/snmp/snmpd.conf` als `root` ohne Passwort ausgeführt werden durfte: `(ALL : ALL) NOPASSWD: /usr/bin/nano /etc/snmp/snmpd.conf`.
    *   `sudo /usr/bin/nano /etc/snmp/snmpd.conf` wurde ausgeführt.
    *   Innerhalb von `nano` wurde mittels `Ctrl+R` (Read File) und `Ctrl+X` (Execute Command) der Befehl `reset; sh 1>&0 2>&0` eingegeben und ausgeführt.
    *   Dies führte zu einer interaktiven Root-Shell.
    *   Die Root-Flag (`2b812156175effb8b80c6a65f8ef3a21`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Broken Access Control (HTTP Header):** Eine Zugriffskontrolle auf eine Webseite konnte durch Manipulation eines HTTP-Headers umgangen werden.
*   **Command Injection (mit Filterumgehung):** Eine Webanwendung war anfällig für Command Injection, wobei Filter durch Base64-Kodierung und Shell-Piping umgangen wurden.
*   **TFTP Information Disclosure:** Ein ungesicherter TFTP-Server gab eine Konfigurationsdatei preis, die einen LessPass-Seed enthielt.
*   **LessPass (deterministischer Passwort-Generator):** Mit dem Seed, der Seite und dem Benutzernamen konnte das Benutzerpasswort generiert werden.
*   **Unsichere `sudo`-Regel (nano):** Ein Benutzer durfte den Texteditor `nano` mit Root-Rechten auf eine spezifische Datei ausführen. Die Befehlsausführungsfunktion innerhalb von `nano` ermöglichte die Eskalation zu Root.
*   **Telnet-Dienst:** Ein unsicherer Telnet-Dienst war aktiv.

## Flags

*   **User Flag (`/home/fanny/user.txt`):** `a8a1d4b692341fc2c59ca33815fb59d6`
*   **Root Flag (`/root/root.txt`):** `2b812156175effb8b80c6a65f8ef3a21`

## Tags

`HackMyVM`, `Oldschool`, `Medium`, `Broken Access Control`, `Command Injection`, `TFTP`, `LessPass`, `sudo Exploit`, `nano Exploit`, `Telnet`, `Linux`, `Web`, `Privilege Escalation`, `Apache`
