---
title: Devvortex
date: 2024-03-26
categories: [HTB]
tags: [htb, joomla, cve]     # TAG names should always be lowercase
image: https://i.ibb.co/LZ9f3Mx/Devvortex.png
---

Siehe auch: [Bevor es losgeht](https://th3t3ngu.github.io/th3t3ngu/Hack-the-Box/)

## Enumeration

Nmap unauffällig. Gobuster unauffällig. Ein Redirect auf Port 80 verrät direkt einen Hostname: `devvortex.htb`.

![enter image description here](https://i.ibb.co/0F4HDmS/page.png)

Die Subdomain-Enumeration von Gobuster findet noch `dev.devvortex`, Gobuster selbst das Directory `/administrator.` Dort wird klar, dass das verwendete CMS Joomla ist:

![enter image description here](https://i.ibb.co/BgVSNh0/joomla.png)

Wie man Joomla enumeriert wird bei [HackTrix](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/joomla) gut beschrieben.

Hat man die Version herausgefunden, weiß man auch, dass Joomla in einer für Information Disclosure angreifbaren Version läuft.

## User
Man kann sich einfach mit curl Informationen von der Box ziehen, unter denen auch Zugangsdaten für den Weblogin sind. Dort angekommen kann man sich unter Einschleusen einer PHP-Reverse-Shell in die Templates einen Shellzugriff auf die Box organisieren - wie das geht wird [hier](https://www.hackingarticles.in/joomla-reverse-shell/) sehr gut beschrieben. 
Problem: Die Templates sind schreibgeschützt - Workaround: Das Template klonen, dann kann man dort beliebig Seiten anlegen. 

Als www-data kann man nicht auf den Benutzer logan und damit auch nicht auf die flag zugreifen, und auch sonst trotz viel stöbern im Webfolder wenig finden. 

Nächste Anlaufstelle: MySQL, das von Joomla als Backend benutzt wird. Auch da funktionieren die Weblogin-Zugangsdaten von lewis. Im Table mit den Nutzerdaten findet sich ein gehashtes Passwort von logan. Diesen Hash kann man mittels [hashes.com](https://hashes.com/en/tools/hash_identifier) einem Typen zuordnen.

Mit John und rockyou hat man so schnell ein Passwort, durch das man sich als logan über SSH einloggen kann.

## Root
Das einzige als root ausführbare Programm ist `apport-cli`, das 2023 Schlagzeilen durch einen Exploit gemacht hat: CVE-2023-1326 - denn es benutzt [less](https://gtfobins.github.io/gtfobins/less/) als Pager. 
Wenn man also einfach einen [Crash generiert](https://wiki.ubuntu.com/CrashReporting) und diesen dann mittels apport-cli ausließt,  erlangt man durch Ausnutzen von less schnell Rootrechte.