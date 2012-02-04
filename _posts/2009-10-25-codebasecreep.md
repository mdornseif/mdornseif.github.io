---
layout: post
title: Codebasekreep
---

{{ page.title }}
================

Irgendwo hier im im Blog hatte ich mal über die Schwierigkeiten, mit einer Codebase geschrieben, von de rman plant, sie die nächsten 30 Jahre zu benutzen. Ich finde den Beitrag aber nicht mehr.

Eine wichtige Prämisse war, dass man die Codebase auf keinen Fall wuchern lassen darf, sonst steigt man spätestens nach 15 Jahren überhaupt nicht mehr durch. Eine der Methoden, mit denen wir dagegen angehen, ist der Schlachtruf &#8220;<a href="http://www.extremeprogramming.org/rules/refactor.html">Refactor Mercilessly</a>&#8220;: wir ändern unseren Code, immer wenn wir ihn besser machen können. Und das können wir dauernd, weil wir ja dauernd bessere Programmierer werden.

Wenn ich einen Blick auf den Code werfe, den wir momentan produktiv einsetzen, ist ein Modul seit 13 Monaten nicht angefasst worden (Steuersoftware für embedded Hardware auf Gabelstaplern), eins seit 10 Monaten (Jahresinventur) und alle anderen wurden vor weniger als drei Monaten &#8211; zumindest teilweise &#8211; überarbeitet.

<img src="http://static.23.nu/md/Pictures/ZZ5A518289.png" alt="" />

Das bedeutet auch, dass wir sehr aktiv Code wegwerfen. Immer mal wieder wird geschaut, wie oft unsere Nutzer eine Funktion überhaupt genutzt haben. Und wenn die Antwort &#8220;selten&#8221; lautet, dann hat die Funktion, gute Chancen, auf nimmerwiedersehen zu verschwinden. Jede Zeile Code in unserem Repository bedeutet Wartungs- und Komplexitätskosten und je weniger Code wir haben, je besser.

Immer wieder ersetzen wir auch selbst gebastelte Funktionalität, durch Fremde Projekte. So ist beispielsweise <a href="http://cybernetics.hudora.biz/projects/wiki/DoDoStorage">DoDoStorage</a> durch <a href="http://blogs.23.nu/c0re/topics/couchdb/">CouchDB</a> ersetzt worden.

<img src="http://static.23.nu/md/Pictures/ZZ657B4B38.png" alt="" />

Ein anderer Trend ist der Weg zu Sprachen und Konstrukten, in denen wir mit weniger Zeilen Code mehr ausdrücken können. Viel davon sind Libraries, bei denen wir immer höhere Abstraktionen für unsere Geschäftsbedürfnisse entwickeln. Aber auch die richtige Sprache für den Job ist ein Thema. Von den mehreren zehntausend Zeilen Ruby Code die wir mal hatten, ist nicht mehr viel übrig. Dafür haben wir in Erlang eine für Lagermanagment spezialisierte Datenbank (myPL/kernelE) geschrieben, die uns Unmengen an Python Code spart.

Aber natürlich ist das alles Tag für Tag ein Kampf. Muss ich wirklich auf Python 2.7/ Django 1.1 updaten? Kann ich den code nicht fürs erste so lassen, wie er ist? Muss die Funktionalität wirklich raus? Ab und zu benutzt sie doch jemand?

Aber wer sagt, dass das Betreuen von Sofwareprojekten was für Feiglinge ist?
