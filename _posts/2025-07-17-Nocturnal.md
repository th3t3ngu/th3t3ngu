---
title: Nocturnal
date: 2025-07-17
categories: [HTB]
tags: [htb, ispconfig, view.php]     # TAG names should always be lowercase
image: https://i.ibb.co/7JMYWfwf/Nocturnal.png
---

Siehe auch: [Bevor es losgeht](https://th3t3ngu.github.io/th3t3ngu/Hack-the-Box/)

## Enumeration

Hier wieder ein Hinweis, der Lebenszeit spart: Ja, die Box ist anfällig für Directory Traversal, aber nein, das allein führt nicht zum Ziel. Die Filterung ist nicht zu umgehen, Schadcode lässt sich so nicht einschleusen, hochgeladene Dateien werden nicht interpretiert - und entgegen der Darstellung auf der Webseite gibts auch keine Backups.

![enter image description here](https://i.ibb.co/V0mMMwvG/Upload.png)

Spannend ist aber trotzdem der Pfad: er zeigt uns nämlich den aktuellen Usernamen an. Probiert man ein wenig herum, dann merkt man schnell, dass dieser nicht an die PHPSESSID gekoppelt ist - der Cookie sorgt nur dafür, dass wir authentifiziert sind, legt aber nicht fest als was - oder wer. 


## User
Doppelt schön: Die Fehlerausgabe ist responsiv, ruft man also einen Pfad mit einem User auf der nicht existiert, wie etwa:

    http://nocturnal.htb/view.php?username=asdasfafag&file=test.pdf

Dann sagt die Box brav "User not found". Ruft man einen Pfad mit einem existierenden Nutzer, aber einer inexistenten (aber den Vorgaben entsprechenden) Datei auf, wie etwa:

    http://nocturnal.htb/view.php?username=tengu&file=nichtda.pdf

Dann sagt uns die Box: Diese Datei existiert nicht, aber hier sind andere Dateien des gleichen Nutzers zum Download.
Es heißt also: Usernamen bruteforcen und hoffen, dass andere Nutzer Dateien hochgeladen haben, die uns weiterhelfen. Jeder Webfuzzer wie [ffuf](https://github.com/ffuf/ffuf) kann das.
Wir finden Usernamen, mit einem davon können wir uns einloggen und stoßen auf eine Mail. Mit dem darin enthaltenen Passwort können wir uns anmelden, ein Backup der Seitenarchitektur erstellen und runterladen.

Das hilft erstaunlich wenig. Wir entdecken in den PHP-files noch einen Hinweis auf einen anderen User, `tobias`, und den relativen Pfad der Database: `../nocturnal_database/nocturnal_database.db.`

Hier nochmal die Info: Auch hier hilft Path Traversal nicht, vergesst es, spart euch eure Lebenszeit.
Des Pudels Kern findet sich hinter der Eingabemaske für das Passwort beim erstellen des Backups: Die ist anfällig für eine [Command Injection](https://hacktricks.boitatech.com.br/pentesting-web/command-injection). Mittels Burps Encoder können wir SQLite3 bitten, die Datanbanken zu dumpen. Wir erhalten Hashes, und da wir durch die Config-Files wissen, dass dr verwendete Hash md5 ist, können wir einige der Hashes knacken. Mit dem erhaltenen Passwort können wir uns per SSH einloggen und haben den User.


## Root
Jap, es ist Ubuntu 20.04 und Sudo in Version 1.8.31, aber weder CVE-2021-3156 noch CVE-2021-3560 funktionieren hier - mein LinPEAS hats auch vorgeschlagen, ich habe es ausprobiert.
netstat zeigt einen ungewöhnlichen, offenen lokalen Port. Piped man sich den mittels SSH auf die eigenen Box, kann man ein [ISPConfig](https://www.ispconfig.org)-Panel öffnen. Die Zugangsdaten von `tobias` passen hier als `admin`. Klickt man sich durch die Gegend, findet man leicht die Softwareversion heraus. Google enthüllt, dass diese angreifbar ist - den Exploit gibts auf Github. Damit kriegt man sehr einfach eine Root-Shell und ist fertig.









