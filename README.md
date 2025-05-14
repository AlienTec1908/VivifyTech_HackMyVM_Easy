# VivifyTech - HackMyVM (Easy)
 
![VivifyTech.png](VivifyTech.png)

## Übersicht

*   **VM:** VivifyTech
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=VivifyTech)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 6. Mai 2024
*   **Original-Writeup:** https://alientec1908.github.io/VivifyTech_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "VivifyTech"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers (Port 80), der eine WordPress-Installation unter `/wordpress/` hostete. Über die WordPress REST API (`/wp-json/wp/v2/users`) wurde der Benutzer `sancelisso` gefunden. Durch Browsen der Verzeichnisstruktur (Directory Listing in `/wp-includes/`) wurde eine Datei `secrets.txt` entdeckt, die eine Wortliste enthielt. Mittels `hydra` und dieser Wortliste wurde das SSH-Passwort (`bohicon`) für den Benutzer `sarah` (Port 22) geknackt. Als `sarah` wurde in `/home/sarah/.private/Tasks.txt` die Credentials `gbodja:4Tch055ouy370N` gefunden. Nach dem Wechsel zu `gbodja` zeigte `sudo -l`, dass `/usr/bin/git` als `root` ohne Passwort ausgeführt werden durfte. Durch Ausführen von `sudo git -p help config` und anschließendem Shell-Escape (`!/bin/sh`) aus dem Pager (vermutlich `less`) wurde eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi`
*   `nmap`
*   `nikto`
*   `dirb`
*   `gobuster`
*   `mysql` (Client)
*   `wpscan`
*   `hydra`
*   `wfuzz`
*   `wget`
*   `cat`
*   `sqlmap` (versucht)
*   `ssh`
*   `sudo`
*   `ls`
*   `id`
*   `find`
*   `getcap`
*   `ss`
*   `uname`
*   `grep`
*   `su`
*   `git`
*   Standard Linux-Befehle (`cd`, `export`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "VivifyTech" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.117`). Eintrag von `vivi.hmv` in `/etc/hosts`.
    *   `nmap`-Scan identifizierte offene Ports: 22 (SSH - OpenSSH 9.2p1), 80 (HTTP - Apache 2.4.57), 3306 (MySQL), 33060 (MySQL X).
    *   `nikto` und `dirb`/`gobuster` auf Port 80 identifizierten eine WordPress-Installation unter `/wordpress/` und Directory Listing für einige Verzeichnisse.
    *   Externer MySQL-Zugriff (Port 3306) war für den Angreifer-Host nicht erlaubt.
    *   WordPress REST API (`/wordpress/index.php/wp-json/wp/v2/users`) offenbarte den Benutzer `sancelisso`.
    *   Ein `wpscan`-Passwort-Bruteforce auf `sancelisso` mit `rockyou.txt` scheiterte.
    *   Ein `hydra`-Angriff auf MySQL Port 33060 mit `rockyou.txt` lieferte wahrscheinlich Falsch-Positive.
    *   LFI-Versuche (`wfuzz` auf `functions.php`) und SQLi-Versuche (`sqlmap`) waren erfolglos.
    *   Manuelles Browsen (Directory Listing) von `/wordpress/wp-includes/` führte zum Fund der Datei `secrets.txt`.

2.  **Initial Access (SSH als `sarah`):**
    *   `cat secrets.txt` zeigte eine Liste potenzieller Passwörter.
    *   Eine manuell erstellte Benutzerliste (`user.txt` mit Namen wie `sarah`) wurde zusammen mit `secrets.txt` für einen SSH-Brute-Force-Angriff verwendet.
    *   `hydra -L user.txt -P secrets.txt ssh://192.168.2.117:22 -t 64` fand die Credentials `sarah:bohicon`.
    *   Erfolgreicher SSH-Login als `sarah`.
    *   User-Flag `HMV{Y0u_G07_Th15_0ne_6543}` in `/home/sarah/user.txt` gelesen.

3.  **Privilege Escalation (von `sarah` zu `gbodja`):**
    *   `sarah` hatte keine `sudo`-Rechte. SUID/Capabilities waren Standard.
    *   `find / -type f -user sarah 2>/dev/null | grep txt` fand `/home/sarah/.private/Tasks.txt`.
    *   `cat /home/sarah/.private/Tasks.txt` enthielt die Credentials `gbodja:4Tch055ouy370N`.
    *   Wechsel zum Benutzer `gbodja` mittels `su gbodja` und dem Passwort `4Tch055ouy370N`.

4.  **Privilege Escalation (von `gbodja` zu `root` via `sudo git`):**
    *   `sudo -l` als `gbodja` zeigte: `(ALL) NPASSWD: /usr/bin/git`.
    *   Versuche, `git` mit `PAGER`-Manipulation auszunutzen, scheiterten, da `sudo` das Setzen der Variable verhinderte.
    *   Erfolgreiche Ausnutzung durch `sudo -u root git -p help config`. Innerhalb des von `git` gestarteten Pagers (vermutlich `less`) wurde `!/bin/sh` eingegeben.
    *   Erlangung einer Root-Shell.
    *   Root-Flag `HMV{Y4NV!7Ch3N1N_Y0u_4r3_7h3_R007_8672}` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **WordPress REST API User Enumeration:** Offenlegung von Benutzernamen (`sancelisso`).
*   **Directory Listing:** Ermöglichte das Auffinden einer `secrets.txt`-Datei mit potenziellen Passwörtern.
*   **Passwort-Brute-Force (SSH):** Erfolgreich mit einer benutzerdefinierten Wortliste.
*   **Klartext-Credentials in Datei:** Zugangsdaten für `gbodja` wurden in einer Textdatei gefunden.
*   **Unsichere `sudo`-Konfiguration (`git`):** Die Erlaubnis, `/usr/bin/git` als `root` ohne Passwort auszuführen, ermöglichte durch einen Shell-Escape aus dem von `git` verwendeten Pager die Erlangung von Root-Rechten.
*   **Shell Escape aus Pager (`less`):** Eine gängige Technik, um aus Pagern, die von privilegierten Prozessen gestartet werden, eine Shell zu erhalten.

## Flags

*   **User Flag (`/home/sarah/user.txt`):** `HMV{Y0u_G07_Th15_0ne_6543}`
*   **Root Flag (`/root/root.txt`):** `HMV{Y4NV!7Ch3N1N_Y0u_4r3_7h3_R007_8672}`

## Tags

`HackMyVM`, `VivifyTech`, `Easy`, `WordPress`, `REST API`, `Directory Listing`, `Password Cracking`, `Hydra`, `SSH`, `sudo Exploitation`, `git`, `Pager Escape`, `Privilege Escalation`, `Linux`, `Web`
