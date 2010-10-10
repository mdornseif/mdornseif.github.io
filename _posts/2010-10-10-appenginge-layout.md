---
layout: post
title: Struktur von AppEngine Applikationen
---

{{ page.title }}
================

<p class="meta">10 Oktober 2010 - Radevormwald</p>

Ich habe nirgends überzeugende Beispiele dafür gefunden, wie man bei Applikationen für [Google Appengine][1]
ein vernünftiges Verzeichnislayout gestaltet. Eine Applikation für AppEngine muss ja sicherstellen, das alle
Abhängigkeiten automatisch hochgeladen werden. Da ich mit in den letzten Wochen vermehrt in der [Dependency Hell][2] wiederfand, sollten die benötigten Libraries bitte automatisch zusammengesucht werden.

In den letzten Monaten setzte ich dabei sehr auf [pip][3], doch die "editable checkouts" von pip bringen einen ganz schnell wieder in die Dependency Hell.

Datei Layout
------------

Die erste Entscheidung war, alle Abhängigkeiten, an denen ich aktiv mitentwickle als [git submodule][4] in `lib` zu packen. Das hat vielerlei Vorteile: Git Submodles referenzieren eine spezifische Revision und werden im Gegensatz zu [svn:externals][5] nicht automatisch geupdated. Das bedeutet, solange ich als Entwickler nicht aktiv eine externe Library update bin ich vor Überraschungen sicher. Es bedeutet auch, dass die Operations Abteilung problemlos genau den gleichen Software Stand für ein deployment erzeugen kann, wie ich als Entwickler auf meinem System habe.

Ein weiterer Vorteil ist, das pip mir versehentlich Änderungen im "editable Checkout" überschreibt oder ich vergesse, eine Änderung zu submitten: Bei Submodulen reicht ein `git status` im Toplevel-Projekt um auch f¨ru die Submodule mitzubekommen, ob dort was uncomittetes liegt.

    $ git status
    #	modified:   lib/DeadTrees (new commits)
    #	modified:   lib/EDIlib (modified content, untracked content)
    #	modified:   lib/huSoftM (modified content)

Der Nachteil ist, dass man abhängigkeiten für diese Subprojekte selber verwalten muss. Das ist aber nicht ganz so schlimm, denn man ist jetzt vor überraschenden Updates sicher. Ich denke momentan nach, ob ich für grösere Software-Projekte nicht lieber von dem jeweiligen Subprojekt einen Branch anlege, um noch sicherer vor überraschenden Änderungen zu sein.

Nun gibt es die eine oder andere Library, an der ich eh nichts ändern werde und/oder die nicht als git repository verfügbar ist. Die sollen weiterhin per pip installiert werden. Dazu screibe ich die gewünschten Pakete in eine Datei Namens `requirements.txt`, die etwa so aussieht:

    Jinja2
    # huTools benötigt httplib2
    httplib2

Per Makefile erzeuge ich ein [virtualenv][6], in das die Pakete mit pip hineien installiert werden. Das ganze sieht dann als `make dependencies` target etwa so aus:

<script src="http://gist.github.com/619112.js?file=Makefile"></script>

Ich bin mir sicher, das man die erzeugung der git Submodule noch eleganter lösen kann, aber es klappt auch so.


Applikationskonfiguration
-------------------------

Jetzt sind alle gewünschten software Module zusammengesucht, was jetzt fehlt ist die Möglichkeit, die Module auch in unserer Applikation zu importieren.

Die Appengine unterstützt die von pip generiertn `.egg-link` Dateien nicht, so dass man die Pfade f¨rud iese Dateien von Hand zusammen suchen muss. Dazu kommen noch die Module, die wir in das `lib` verzeichnis gelegt haben. Um es spannender zu machen, kann das aktuelle Verzeichnis, aus dem unsere Skripte aufgerufen werden bei der AppEngine je nach Lust und Laune variieren. Alles in allem kommt daher ein recht komplexes Pfad-Konfigurationsskript dabei heraus.

In der Datei `config.py` werden neben den Pfaden auch direkt die gewünschte Django Version und das Verzeichnis für die Templates gesetzt. Im Ergebnis sieht das dann so aus:


<script src="http://gist.github.com/619112.js?file=config.py"></script>

Was sich dabei herausgestellt hat, ist dass das Setzten von Environment-Variablen (hier `PYPOSTAL_PIXELLETTER_CRED`) in `config.py` keine gute Idee ist. Ob es am Module-Caching der AppEngine liegt, oder woran auch immer, gelegentlich findet man in dem Modul, wo amn sie ausliest, die Environment Variablen leer. [pyJasper][7] ist deswegen schon auf einen alternativen Konfiguartionsmechanismus umgestiegen, [pyPostal][8] hat das noch vor sich.

Nun kann man einfach in jedem Modul der eigenen Applikation als erstes `config.py` importieren und man ist fast im grünen Bereich. Vermutlich bruacht man noch Sessions und dergleichen, so dass ein `appengine_config.py` file fällig wird, in dme auf jeden Fall auch `config.py` importiert werden muss.

<script src="http://gist.github.com/619112.js?file=appengine_config.py"></script>

Zu guter Letzt sollte man noch dafür sorgen, dass bei einem Deployment all das Zubehör zu den Libraries nciht mit auf den Server geladen wird. Das kann in der `app.yaml` dann z.B. so aussehen:

<script src="http://gist.github.com/619112.js?file=app.yaml"></script>

[1]: http://de.wikipedia.org/wiki/Google_App_Engine
[2]: http://en.wikipedia.org/wiki/Dependency_hell
[3]: http://pypi.python.org/pypi/pip
[4]: http://speirs.org/blog/2009/5/11/understanding-git-submodules.html
[5]: http://svnbook.red-bean.com/en/1.0/ch07s03.html
[6]: http://pypi.python.org/pypi/virtualenv
[7]: http://pypi.python.org/pypi/pyJasper
[8]: http://pypi.python.org/pypi/pyPostal

