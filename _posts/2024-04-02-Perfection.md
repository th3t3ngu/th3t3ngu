---
title: Perfection
date: 2024-04-02
categories: [HTB]
tags: [htb, ssti]     # TAG names should always be lowercase
image: https://i.ibb.co/R3xnpR9/Perfection.png
---

Siehe auch: [Bevor es losgeht](https://th3t3ngu.github.io/th3t3ngu/Hack-the-Box/)

## Enumeration

Nmap unauffällig, Gobuster unauffällig. 

Die Homepage verweißt auf einen Calculator zum berechnen von Studienleistungen.:

![enter image description here](https://i.ibb.co/NLBy55V/calc.png)

Testet man nach [SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection), dann merkt man, dass hier eine Eingabekontrolle läuft - die Webseite scheint bestimmte Sonderzeichen zu blocken, was das Ausführen von Code herausfordernd macht.

![enter image description here](https://i.ibb.co/tpZZhN6/malicious.png)

## User
Hier hilft vor allem: Rumprobieren, was geht und was nicht. Relativ schnell landet man bei URL-Encoding (Burp hat einen eigenen Tab dafür): Die Webseite dekodiert selbstständig URL, aber: Der dekodierte Code wird vor dem Ausführen nochmal überprüft, also so einfach überlistet man die Eingabekontrolle nicht.

Schlussendlich kam zumindest mir der Zufall zu Hilfe, in Form eines Fehlers: Auf der Tastatur abgerutscht habe ich einen Zeilenumbruch generiert. Und dieser Zeilenumbruch scheint den Unterschied zu machen - alles was in der neuen Zeile steht wird von der Eingabekontrolle nicht beachtet:
![enter image description here](https://i.ibb.co/CnT0xjc/code.png)

Jetzt kann man nach der verwendeten Sprache testen, die im Backend benutzt wird (hierbei hilft wieder [HackTrix](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection) enorm) und sich so eine Reverse Shell basteln. Lezteres war bei mir ein wenig knifflig - bash wollte nicht durchgehen, Ruby meldete sich zurück, brach aber sofort ab. Verbindungsprobleme. 
Als Workaround kam wieder zum Einsatz: Shell schreiben, speichern, auf eigenem Rechner hosten, die Box die Shell mittels curl ziehen lassen und dann zu bash pipen. Damit hat man eine Shell und auch die erste flag.

## Root
Im Ordner Migration findet sich `pupilpath_credentials.db`, das man mit sqlite3 öffnen und eine Liste von Namen samt Hashes bekommt. 

Der Hash lässt sich mit dem [Hash Analyzer](https://hashes.com/en/tools/hash_identifier) einem Typen zuordnen... und keine Wordlist funktioniert. 

Nach viele Gesuche findet sich des Rätsels Lösung in /var/mail/susan: Dort wird eine neue Passwortregelung vorgeschlagen, an die sich scheinbar alle brav gehalten haben:

    {firstname}_{firstname backwards}_{randomly generated integer between 1 and 1,000,000,000}

So kann das Passwort ja auch in keiner Wortlist stehen. Damit ist das Passwort susan_nasus_???, wobei ??? für eine Zahlenfolge zwischen 1 und 1,000,000,000 steht. Das kann hashcat leicht herausfinden. Da susan in den Sudoers eingetragen ist, wird man mit dem Passwort direkt root und holt die flag.



