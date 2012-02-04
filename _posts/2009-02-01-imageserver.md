---
layout: post
title: Imageserver
---

{{ page.title }}
================

Aus einem internen Blogposting:

Ein Großteil unserer Preformanceprobleme vom Freitag hingen mit dem Weg zusammen, wie wir nicht Statische Bilder (insbesondere Produktbilder) verwalten. Bisher haben wir mehr oder weniger unverändert die Django Möglichkeiten Verwendet. Das bedeutet, ein hochgeladenes bild wird im lokalen Filesystem gespeichert und der Pfad zur Datei wird in der Datenbank abgelegt. Das klappt natürlich immer nur auf dem Rechner, auf dem die Dateien auch gespeichert sind (Probleme beim Testen).

Wenn ein Bild vom Browser angefordert wird, kann der Apache dies direkt zurückliefern, ohne dass eine Zeile Python Code ausgeführt werden muss.

Darauf aufsetzend hatten wir das `ScalingImageField`. Das konnte zum einen direkt fertige Image Tags (&lt;img src=&#8221;http://foo.bar.de/media/,/bla.jpg&#8221; alt=&#8221;Bigwheel&#8221; height=&#8221;480&#8243; width=&#8221;640&#8243;>) erzeugen, zum anderen auch die bilder nach Wunsch skalieren.

Das ganze wurde mit allerlei Meta-Klassen Magie an Django&#8217;s imgaefield angeflanscht. Magie bei Seite, konnte man sagen, &#8220;gib mir die URL von diesem Bild in der Größe &#8216;thumbnail&#8217;&#8221; und kriege eine URL, wo der Thumbnail zu finden war. Dazu wurde der Thumbnail direkt erzeugt, wenn z.B. ein Template nach der URL des Thumbnails fragte. Das Sorgte dafür dass dann, wenn der Browser am Ende die URL abrief, die Datei direkt vom Apache ohne weitere Eingriffe von Python abgerufen werden konnte.

Alles schön und gut, aber: Durch einen Programmierfehler (cklein war als lettzter dran, die Grundsteine dafür hatte ich aber schon mit der Django 1.0 Umstellung gelegt. Nun wurde jedes mal, wenn ein Thumbnail (oder eine andere skalierte Bildgröße) angefordert wurde ein neues <em>Verzeichnis</em> angelegt und der Thumbnail neu berechnet. Das führte zum einen dazu, dass der Server durch das viele Berechnen langsamer wurde, zum anderen war bald die maximale Anzahl von Unterverzeichnissen in einem Verzeichnis (32000) erreicht, so dass dann Schluss mit lustig war.

Bug gefixed, die ganzen Verzeichnisse mit den skalierten Bildern gelöscht (die Originale haben wir natürlich noch behalten) und alles ist gut &#8211; dachten wir. Leider hatte der Server dann schnell eine Load von knapp 30 und nichts war gut.

Was war passiert? Wir haben Seiten, auf denen sind hunderte von links zu Bildern &#8216;drauf (z.B. http://www.example.com/katalog/katalog-2009/bilder/). Nun werden die Bilder ja bereits skaliert, wenn der <em>Link</em> zu dem Bild durch das Template angefordert wird. Wenn also ein Web Crawler (der sich gar nicht für Bilder Interessiert) das HTML con http://www.example.com/katalog/katalog-2009/bilder/ abruft, beginnt das Template alle Artikelbilder im Katalog zu skalieren. Dem Crawler dauert das natürlich zu lange, er schliesst die Verbindung. Der Webserver rendert aber munter weiter die Seite, indem er alle Bilder skaliert.

Der Crawler versucht derweil vielleicht erneut, die Seite abzurufen. Da sich die skalierten Bilder von dem ersten Aufruf noch nicht auf der Platte befinden wird erneut begonnen, alle Bilder zu skalieren. Usw. Und das, obwohl der Webserver kein einziges Bild hätte ausliefern müssen! Schliesslich wurde nur HTML abgerufen.

Da muss was besseres her. Anforderungen sind:

* Darf nicht an einen einzelnen Rechner gebunden sein
* Bilder werden erst gerendert, wenn sie gebraucht werden
* Bei Bedarf sollten mehrere Rechner Bilder parallel rendern können
* Mindestens so schnell wie die jetzige Lösung unter Normallast
* Bei Übrlastung &#8220;fail gracefully&#8221;.

Dabei rausgekommen ist der huDjango Imageserver, von dem ich einen Prototypen gebastelt habe.

Erste Designentscheidung ist, dass Bilde ausschließlich über zufällige 160 Bit Nummern identifiziert werden. Wenn man ein Bild speichert, bekommt man einen ID zurück. Dass Bild kann dann nur noch mit diesem ID angesprochen werden. Wenn man den ID nicht weiss, ist das Bild &#8220;weg&#8221;. Vorteil davon ist, dass wir uns keine Gedanken um Passwortschutz etc. von internen Bildern machen müssen. Wer den ID nicht kennt, kommt nicht an das Bild.</p>
<p>Zweite Designentscheidung: Man kann ein Bild nicht ändern. Alte Bilder bleiben im System, wenn man ein geändertes Bild hochläd, bekommt man einen neuen ID zurück. Das vereinfacht Caching enorm.

Dritte Desingentscheidung: wir speichern die Originalbilder in CouchDB.  Aufwand und nutzen stehen dabei in einer sinnvollen Relation.

Wenn man ein Bild in dem System speichert wird de rID des bildes basierend auf SHA1 berechnet, und das bild unter diesem ID in die CouchDB gespeichert.

Der in Python geschriebene Imageserver parst einen Pfad wie `/o/22N6MZ7BULQP4UYKWGU5LDQ4DI2DK5HH01.jpeg` zunächst in siene Bestandteile: die Endung wird weggeschmissen, der &#8220;Dateiname&#8221; ist die Bild-ID und das &#8220;Verzeichnis&#8221; die gewünschte Bildgröße (&#8221;o&#8221; steht für Origninal). Wenn sich das Bild bereits auf der Lokalen Festplatte gecached befindet, wird es direkt zum Browser geschickt. Ansonsten wird aus CouchDB das Bild abgerufen, in den Cache gepackt und dann zum Browser geschickt.  Wenn es sich um einen Skalierungswunsch handelt, dann wird erst das Original abgerufen, gecached, dann umgerechnet, das wiederum gecached und dann an den Browser gesendet.

Auf http://images.example.com/ ist das ganze in einer Mixtur aus Python, FastCGI und Lighttpd intalliert. Dsa ganze hat da noch einen raffinierten Trick: Lighttpd liefert direkt aus dem Cache aus, hat aber als 404-handler den Python server installiert. D.h. der Python server wird nur aufgerufen, wenn die Datei nicht schon gecached ist.

Das kann problemlos auf mehreren Rechnern installiert werden und hat eine sehr adäquate Performance.

Was jetzt noch fehlt ist die Django Anbindung: anstatt Bilder im Filesystem zu speichern, sollten Sie direkt auf dem Image Server landen.
