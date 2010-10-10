---
layout: post
title: Paginierung auf AppEngine
---

{{ page.title }}
================

<p class="meta">2 Oktober 2010 - Raeren</p>

Für [Google Appengine][1] zu entwickeln hilft einem sehr, einige Teile von Webanwendungen schon in der Entwicklung richtig hinzubekommen. So fängt die Appengine sofort an zu motzen, wenn eine Anfrage mehr als 1 Sekunde zur Verarbeitung benötigt. Generell wei smal na, das Webseiten schnell sein sollten, aber man ist doch immer wieder verführt, zu viel in einem einzlenen Request zu erledigen.

Es gehört ja  auch durchaus zu den Prinzipien des Webs (und insbesondere dem [RESTful][2] API-Design), das man daten in relativ kleine Häppchen aufteilt und "Blätterns" implementiert. Das war übrigends eine Ziemliche Revolution, als die ersten Suchmashcienen, das Seitenbasierte Konzept des "Blätterns" von den schwerrfälligen Mainframe-Betriebssystemen. Weder dem Betriebssystem des Internets (Unix) noch dem Webbrowser ist blätterrei  eigen.

Nichts des do trotz hat sich blättern als das Tool der Wahl zum Darstellen und Übertragen grosser Datenbestände etabliert. Man kann sogar sagen, dass das Konzept über [Generator-Konstrukte][3] vom Internet in die Programmiersprachennutzung geschwappt.

![Paginierung mit zu vielen Seiten](/images/pagination_130.png)

Als Visuelles Element der Paginierung hat sich am Ende der Seite ein _Vor-Button_ und ein _Zurück-Button_ etabliert. Gerne wird auch noch für jede Seite, auf die geblättert werden kann, ein direkter Link angezeigt. Das führt aber oft zu Grotesken und wenig hilfreichen Ergebnissen, wir oben gezeigt. Bei drei oder vier Seiten mögen diese Direktlinks noch Sinn machen, aber bei duzenden von SEiten, reicht mir eigentlich die Information, dass es _sehr viele_ Seiten gibt. Davon eine direkt anzuspringen, macht selten Sinn.

Die Lösung bei vielen Webdesigns ist es, nur Direktlinks für die ersten zehn Seiten anzuzuegen.

![Paginierung bei Google](/images/google_pagination.png)

Aber selbst das macht oft keinen Sinn. In vielen Zusammenhängen ist die Zahl "vorherigen" Elemente in der Praxis unendlich: Zeitungsartikel, Suchergebnisse für "gratis", Fotos von der Golden gate Bridge usw. Da macht es auch Sinn, das User Interface für das Bluattern auf "vor" und "zurück" zu beschränken.

Unser aktuelles Beispiel sind Rechungen eines Kunden. Wenn der Kunde ein guter und langjärigre Kunde ist, dann reichen seine Rechungen ja auch sehr weit (90 Jahre, wenn auch nicht komplett elektronisch verfügbar) zurück. Was aber vor allem Interessiert, sind die letzten Rechungen. Warum also Resourcen darein investieren, herauszufinden, wie viel Rechungen insgesammt für einen im System sind? Das interessiert in der regel nicht. Hier reicht ganz klar eine Funktion um "ältere Rechungen" anzuzeigen.

![vor und zurueck](/images/zurueck_vor.png)

Das Userinterface hat weniger potentiell verwirrende Elemente und man bruacht weniger Rechenzeit.

[1]: http://de.wikipedia.org/wiki/Google_App_Engine
[2]: http://www.oio.de/public/xml/rest-webservices.htm
[3]: http://de.wikipedia.org/wiki/Iterator#Generatoren


Implementierung auf der Appengine
---------------------------------

Im folgenden ein "Controller-" und ein HTML Schnipsel, wie man das Ganze auf der Appengine implementieren kann. Erstmal brauchen irgendein Datastore Model, das wir sortiert darstellen können - denn mblettern macht nur in einer sortierten Menge von Datensätzen Sinn. Im Beispiel ist das `models.Rechnung` und wir sortieren absteigend nach `leistungsdatum`.

Wir akzeptieren beim Aufruf der Seite die Parameter `start`, `count`, deren Bedeutung klar sein sollte: Was ist der erste Datensatz den wir darstellen udn wie viele sollen dargestellt werden. Die Appengine Funktion `get_range()` sorgt dafür, das unsere Nutzer nicht irgendwelche wilden Traumwerte übergeben können.

Nun holen wir uns einen Datensatz mehr, als vom user Verlangt wurde. So können wir feststellen, ob noch weiter eDatensätze folgen, ohne eine zweite Query machen zu müssen. In `more_objects` und `prev_objects` wird gespeichert, ob man vor- und rückwärts blättern kann und `next_start` sowie `prev_start` geben an, wie der `start` PArameter für die nächte und die vorherige Seite sein muß.

<script src="http://gist.github.com/607757.js"> </script>

Im zugehörigen HTML sind vermutlich die Unicode-Pfeile das aufregenste. Das Beispiel sollte sich sowohl mit Django Templates als auch mit Jinja2 rendern lassen.