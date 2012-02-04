<h2 class="entry-title">Webseitenoptimierung</h2>

Nachdem wir Ende letzter Woche einen mächtigen Performance Einbruch unseres Webservers hatten, hab ich mir mal etwas detailliertere Gedanken über die Geschwindigkeit unserer Webseite gemacht.

Die Webseite brauchte gut eine Sekunde zum laden. Mein Ziel (aus rein sportlichen Gründen) war es diese zeit zu halbieren. Hat gekappt:

<img src="http://static.23.nu/md/Pictures/ZZ04906ADB.jpg" width="480" height="133" alt="" />

Was habe ich gemacht?

Zunächst habe ich alle statischen Dateien (CSS, GIF) auf einen separaten Webserver (<a href="http://www.lighttpd.net/">lighttpd</a>) verschoben. Dieser wurde auf statische Inhalte optimiert, so dass keine E-Tags erzeugt werden, `Cache-Control: expires` erzeugt werden und CSS on the fly komprimiert wird.

Dann hab ich allerlei Javascript tracking code aus der Webseite entfernt, den wir eh nicht mehr nutzen.

Als letztes habe ich einen CSS/JavaScript optimizer verwendet, um die Größe der CSS Dateien um nochmals 25% zu verringern.

Das ging einfach.
