---
title: Environment
date: 2025-07-22
categories: [HTB]
tags: [htb, laravel, env, bypass]     # TAG names should always be lowercase
image: https://i.ibb.co/jPQXKTsP/Environment.png
---

Siehe auch: [Bevor es losgeht](https://th3t3ngu.github.io/th3t3ngu/Hack-the-Box/)


## Enumeration

Ich habe schon lange nicht mehr so eine Vektor-Orgie gesehen - auf dieser Box gibt es so viele unterschiedliche Dinge die man anschauen kann, einfach herrlich!
Schlagen wir mal eine Schneise in das Dickicht und merken uns die vielen unterschiedlichen Ansatzpunkte: 

1) Direkt auf der Homepage werden wir von einem grünen Stockfoto begrüßt und bekommen die thematische Einführung: Wir sind bei einer Umweltstiftung gelandet. Ganz weit unten finden wir zwei interessante Dinge: Die Aufnahme in die Mailingliste funktioniert tatsächlich, wir bekommen die Nachricht, dass unsere Mailadresse gespeichert wurde. Es gibt also ein verarbeitendes Backend. Und: `Production v1.1`

Unser [ferox](https://github.com/epi052/feroxbuster) müsste mittlerweile schon einiges zutage gefördert haben, unter anderem:

2) `/login` bringt uns zum Weblogin vom Marketing Management Portal. Wir merken uns hier vor allem Marketing.

3) unter`/up` gibt es ein kleines [tailwind](https://tailwindcss.com)-snippet, dass darüber Auskunft gibt,  dass eine nicht näher genannte "Application" gerade läuft.  Es sagt auch, dass es einen HTTP-Request empfangen hat. Waren wir das?

4) `/upload` und `/mailing` sind [Ignition](https://github.com/spatie/laravel-ignition)-Errorpages und gleichen sich in der Hinsicht, dass sie beim Aufrufen einen 405 werfen und uns sagen, dass GET nicht erlaubt ist, nur POST. Zusätzlich bekommen wir noch eine PHP- und [Laravel](https://github.com/laravel/laravel)-Version, und irgendwie scheint auch [Symfony](https://github.com/symfony/symfony) mit drin zu hängen. 
Laravel in dieser Version ist [angreifbar](https://muneebdev.com/laravel-11-30-0-exploit/). 

## Foothold
Ich habe mich entschieden, als erstes mit (4) anzufangen: Wir kriegen Session-Cookies von Laravel und der CVE legt nahe, dass wir durch den Debug-Mode irgendwie einen [XSS-Inject](https://hacktricks.boitatech.com.br/pentesting-web/xss-cross-site-scripting) hinbekommen sollten. 
Das klappt auch, der XSS wird reflektiert - aber da keine Admin-Session aktiv ist, kriegen wir keinen Cookie, und damit keinen Zugang.

Nächster Versuch: Ignition. Diese Errorpages sind anfällig für eine Backdoor, die RCEs ermöglichen, sobald man auf deren Endpoints zugreift. 
![enter image description here](https://i.ibb.co/LDMJ4b6j/Upload.png)
Die liegen normalerweise unter `_ignition/solutions`. Hier nicht. Viel Gesuche, angepasste Ferox-Wordlists brachten: nichts. Das ist das Problem mit Vektor-Orgien.

`/up` stellt sich als einfaches Snippet dar, das standardmäßig mit Laravel ausgeliefert wird und anzeigt, dass die Seite läuft. Auch hier also kein Glück.


Zuletzt: Die Loginform unter `/login`.  Hier passiert erstmal wenig spannendes, bis etwas auffällt, das sonst meistens fehlt: `Remember Me?` 
![enter image description here](https://i.ibb.co/Q34RD0j7/MMM.png)
Klickt man das Kästchen an, wird es als POST in der Form `remember=True` an die Box übergeben. Das Argument ist binär: True, False - alles andere wirft einen Fehler. Und der kommt wieder in der Form einer Ignition-Errorpage daher, aber diesmal einer hilfreichen. 
![enter image description here](https://i.ibb.co/5gsg87qn/env.png)
Wir sehen hier jede Menge nützliches und mit den neuen Informationen finden wir schnell die Informationen die wir brauchen: Laravel in dieser Form ist angreifbar für eine [Enviornment manipulation](https://www.cybersecurity-help.cz/vdb/SB20241112127) - damit ist auch der Name der Box erklärt. 
Nutzt man diese Lücke aus, bekommt man eine privilegierte Session und wird automatisch zu `/management/dashboard` weitergeleitet. Dort können wir nicht viel machen, außer unser Profilbild zu ändern - und vielleicht eine Reverse Shell darin zu verstecken. Damit haben wir endlich eine RCE und eine Shell auf der Box. 


## User
Wir landen als `www-data` auf der Box und können uns umschauen. Die user.txt im Homeverzeichnis können wir zwar lesen, aber  Zugangsdaten zum User haben wir immer noch nicht. Dafür finden wir eine verschlüsselte `keyvault.gpg` unter `/backup`. Um die zu entschlüsseln, helfen uns auch wieder *enviromental variables*: Zugriff auf `/home` haben wir natürlich nicht, dafür auf `/tmp`. Wenn wir dort einen Ordner anlegen, die Zugangsberechtigungen rekursiv ändern und dann die `keyvault.gpg` dort reinkopieren, können wir mit gpg einfach die Schlüssel extrahieren und die Datei entpacken. Drinnen finden wir allerhand peinliche Passworte, auch für den SSH Zugang. Damit sind wir endlich user.


## Root
sudo -l liefert uns auch hier wieder das Spielzeug, mit dem wir umgehen dürfen. Wir haben Zugriff auf jede Menge Variablen, darunter - natürlich - wieder *enviroment*. Wenn wir `path` und `env` klug einsetzen, haben wir sehr schnell eine rootshell auf der Box und sind damit fertig. 