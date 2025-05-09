# Breakout - HackMyVM Writeup

![Breakout VM Icon](Breakout.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "Breakout" (Schwierigkeitsgrad: Easy), erstellt von DarkSpirit. Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** Breakout
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Easy
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Breakout](https://hackmyvm.eu/machines/machine.php?vm=Breakout)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 24. April 2023
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/Breakout_HackMyVM_Easy/](https://alientec1908.github.io/Breakout_HackMyVM_Easy/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die Breakout-Maschine umfasste folgende Schritte:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.136` unter dem Hostnamen `breakout`) mittels `arp-scan`.
    *   Ein `nmap`-Scan offenbarte mehrere offene Ports: HTTP (80, Apache), NetBIOS/SMB (139/445, Samba 4.6.2), und zwei Webmin-Instanzen auf Port 10000 (HTTPS, MiniServ 1.981) und Port 20000 (HTTP, MiniServ 1.830).
2.  **Web Enumeration & Samba:**
    *   `gobuster` auf Port 80 fand das Verzeichnis `/manual`.
    *   `enum4linux` auf Samba (Null-Session erlaubt) identifizierte den Benutzer `cyber` und eine schwache Passwortrichtlinie (Mindestlänge 5, Komplexität deaktiviert).
    *   Ein Login-Versuch auf `http://192.168.2.136:10000/` zeigte, dass SSL erforderlich ist. Der Fokus lag daher auf Port 20000 (HTTP).
    *   Ein `hydra`-Brute-Force-Versuch auf Webmin (Port 20000) mit dem Benutzer `cyber` und `rockyou.txt` war (implizit) erfolglos.
    *   Im Quellcode einer Webseite (vermutlich Port 80 oder Webmin) wurde ein HTML-Kommentar mit Brainfuck-Code gefunden, der als "verschlüsselte Zugangsdaten" beschrieben wurde.
    *   Die Entschlüsselung des Brainfuck-Codes ergab das Passwort `.2uqPEfj3D P'a-3`.
3.  **Initial Access (Webmin):**
    *   Erfolgreicher Login bei der Webmin-Instanz auf Port 20000 (vermutlich `https://breakout:20000/` oder HTTP, je nach korrekter Interpretation des Logs) mit den Anmeldedaten `cyber` / `.2uqPEfj3D P'a-3`.
    *   Über die integrierte Terminal-Funktion von Webmin wurde Befehlszugriff als Benutzer `cyber` erlangt.
    *   Die User-Flag (`/home/cyber/user.txt`) wurde gelesen.
    *   Eine Reverse Shell wurde vom Webmin-Terminal zum Angreifer-System aufgebaut und stabilisiert.
4.  **Privilege Escalation Preparation:**
    *   Im Home-Verzeichnis von `cyber` befand sich eine `tar`-Datei, die `root` gehörte, aber für `cyber` ausführbar war.
    *   `getcap tar` zeigte, dass diese spezifische `tar`-Datei die Linux Capability `cap_dac_read_search=ep` besaß. Dies erlaubt das Umgehen von Datei-Leseberechtigungen.
    *   Im Verzeichnis `/var/backups/` wurde eine versteckte Datei `.old_pass.bak` gefunden, die `root` gehörte und nur für `root` lesbar war.
5.  **Privilege Escalation (via Tar Capability):**
    *   Die `tar`-Datei mit der `cap_dac_read_search`-Capability wurde verwendet, um die Datei `/var/backups/.old_pass.bak` in ein neues Archiv (`bak.tar`) im Home-Verzeichnis von `cyber` zu packen: `./tar -cf bak.tar /var/backups/.old_pass.bak`.
    *   Aus dem Inhalt der `bak.tar`-Datei wurde das Passwort `Ts&4&YurgtRX(=~h` extrahiert.
    *   Mit `su root` und dem Passwort `Ts&4&YurgtRX(=~h` wurde erfolgreich Root-Zugriff erlangt.
6.  **Flags:**
    *   Die User-Flag wurde als `cyber` gelesen.
    *   Die Root-Flag (`/root/rOOt.txt`) wurde als `root` gelesen.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `enum4linux`
*   `hydra` (Versuch)
*   Brainfuck Interpreter (extern)
*   `curl`
*   `nc` (netcat)
*   `python3` (pty)
*   `stty`
*   `find`
*   `getcap`
*   `ls`
*   `cd`
*   `cat`
*   `tar`
*   `su`
*   `id`
*   Web Browser
*   Webmin (als Ziel)

## Identifizierte Schwachstellen (Zusammenfassung)

*   **Exponierte Samba-Informationen:** Null-Sessions erlaubten die Enumeration von Benutzern (`cyber`) und Passwortrichtlinien.
*   **Informationspreisgabe durch HTML-Kommentare:** Ein Passwort (`.2uqPEfj3D P'a-3` für `cyber`) wurde in Brainfuck-Code in einem HTML-Kommentar "verschlüsselt".
*   **Veraltete Webmin-Instanz (Version 1.830):** Bot eine Angriffsfläche, obwohl der Exploit-Pfad über gefundene Credentials lief.
*   **Unsichere Speicherung von Passwörtern in Backup-Datei:** Ein (altes) Root-Passwort (`Ts&4&YurgtRX(=~h`) wurde in `/var/backups/.old_pass.bak` gespeichert.
*   **Gefährliche Linux Capability (`cap_dac_read_search` auf `tar`):** Eine lokale `tar`-Datei im Home-Verzeichnis von `cyber` hatte die `cap_dac_read_search`-Capability, die es erlaubte, trotz fehlender Berechtigungen auf die Passwort-Backup-Datei zuzugreifen und somit das Root-Passwort zu exfiltrieren.

## Flags

*   **User Flag (`/home/cyber/user.txt`):** `3mp!r3{You_Manage_To_Break_To_My_Secure_Access}`
*   **Root Flag (`/root/rOOt.txt`):** `3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}`

---

**Wichtiger Hinweis:** Dieses Dokument und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Techniken und Werkzeuge sollten nur in legalen und autorisierten Umgebungen (z.B. eigenen Testlaboren, CTFs oder mit expliziter Genehmigung) angewendet werden. Das unbefugte Eindringen in fremde Computersysteme ist eine Straftat und kann rechtliche Konsequenzen nach sich ziehen.

---
*Bericht von Ben C. - Cyber Security Reports*
