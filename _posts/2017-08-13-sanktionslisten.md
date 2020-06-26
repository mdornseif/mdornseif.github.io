---
layout: post
---

# Compliance: Sicherstellen, dass man Sanktionslisten beachtet.

Als gewissenhafter Kaufmann ist man verpflichtet, _sehr_ viele Vorschriften zu beachten. 
Zum Beispiel gibt es mannigfache _Sanktionslisten_ mit denen die EU, die UN und allerlei Regierungen verbieten mit bestimmten Personen, Gruppierungen und Staaten Geschäfte zu machen.

Nun ist es sicher gut und richtig, dass unsere Banken für Leute wie dem syrischen Major General Mohamed Khaddor kein Konto eröffnen und das keiner so ohne weiteres an 
Andrei Valeryevich Katapolow (der auf der russischen Seite mächtig die Finger in der Krim-Besetzung hatte) Panzerhaubitzen liefert. 

Wenn man aber einen Webshop mit <a href="http://lahengst.com/die-braut-haut-ins-auge/#more-339">Die Braut haut ins Auge</a> Bootlegs und T-Shirts betreibt, klingt das ganze wie Gängelei.

Naja, Umsatzsteuervoranmeldung macht auch keinen Spaß, also gilt es, die gesetzlichen Anforderungen mit möglichst wenig Aufwand und viel Eleganz zu erfüllen. Problem: Wie vieles Zwischenstaatliches Recht gibt es auch bei den Sanktionslisten kein wirkliches Gesetz, in dem sich findet, was man denn nun zu tun hat und was nicht. Das ist alles ehr mündlich tradiert. Auf jeden Fall will der Zoll bei einer Zollprüfung und spätestns bei einem [AEO-Audit](https://www.zoll.de/DE/Fachthemen/Zoelle/Zugelassener-Wirtschaftsbeteiligter-AEO/zugelassener-wirtschaftsbeteiligter-aeo_node.html) ein Häkchen bei "Sanktionslisten" machen.

Da gibt es teure Software- und Beratungspakete zu, die einem dabei helfen sollten. Ich hab in einem früheren Leben man mitwirken dürfen, solche Machwerke zu analysieren, penetrieren und zertifizieren und ... nunja, ich würde kein Geld für solche Software ausgeben.

Gut; was ist zu tun: <b>Keine Geschäfte mit Leuten auf der Sanktionsliste machen.</b>

Also Sicherstellen, dass:

* In Kreditoren, Debitoren und Mitarbeiterstammdaten keiner der auf den Listen stehenden Personen vorkommt. 
* Das in Transaktionsdaten ("Einmaladressen") keiner der auf den Listen stehenden Personen vorkommt. 
* Das aktuellen Listen verwendet werden.


Die Sanktionslisten, die es so gibt [sind zwar ganz schön unübersichtlich](https://sanctionsmap.eu/#/main), aber zum Glück kann man das ganze [aggregiert und Maschinenlesbar von der EU herunter laden](https://data.europa.eu/euodp/en/data/dataset/consolidated-list-of-persons-groups-and-entities-subject-to-eu-financial-sanctions/resource/3a1d5dd6-244e-4118-82d3-db3be0554112) tolle Sache.

Also kann man recht einfach, das ganze implementieren:

1. Liste zB wöchentlich runterladen, parsen, in ein datenbankfreundliches Format umwandeln. Protokollieren, was man da runtergeladen hat, wie das Parsen geklappt hat.
2. Liste gegen Stammdaten matchen. Anzahl der überprüften Datensätte, von wem wann die Liste runtergeladen wurde, etc. protokollieren. Bei Matches einen Menschen informieren, sicherstellen, dass auch dieses protokolliert wird. Der Mensch muss dann protokollieren, was er für schritte unternommen hat.
3. Vielleicht eine Liste von False Positives, die erlaubt sind, führen. Deren Länge und wer da was einfügt natürlich protokollieren.

Und wie soll der Vergleich, ob da jemand auf der Liste steht, abgehen? Mit Soundex und Fuzzy Matching? Nein! Ich würde Argumentieren, dass wir uns schon darauf verlassen dürfen, dass alle Schreibweisen und Pseudonyme von den zuständigen Stellen nach besten Wissen und Gewissen auf die Liste gesetzt werden. Wenn da <code>Maximi<b>ll</b>ian Dornseif</code> auf der Liste steht, dann dürfen wir mit <code>Maximi<b>l</b>ian Dornseif</code> weiter Geschäfte machen. Wird schon seinen Grund haben. Denn mit genug Fuzzy matching kann ja praktisch je nach Parameter jeder Name matchen.

Übererfüllugn von Vorschriften verlangt ja keiner. Wenn auf der Autobahn Tempo 80 steht, fahr ich ja auch nicht vorsichtshalber 65 in vorauseilendem Gehorsam. Auch wenn Leute, deren übervorsichtiges Geschäft “Compliance” ist das gerne mal fordern.

Ein Beispiel für ein "Sanktionslisten as a Servie" findet sich in <em>Sanctext</em>, einer Software, die zunächst auf Django mit einer SQL Datenbank entwickelt wurde und dann auf die Google App Engine mit dem Datastore portiert wurde. Schon etwas angegraut, zeigt aber ganz gut, wie so etwas gehen kann.

Funktionierenden Programmcode zum Thema gibt es unter [https://github.com/mdornseif/sanctex](https://github.com/mdornseif/sanctex).
