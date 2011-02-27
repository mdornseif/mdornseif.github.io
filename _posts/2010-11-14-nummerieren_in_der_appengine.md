---
layout: post
title: Duchzählen auf auf der AppEngine
---

{{ page.title }}
================

Ich wollte mich daran machen, unsere Logistik Applikation [huLOG][1] auf die Google AppEngine zu portieren. Die Applikation greift kaum auf externe Systeme zu, und macht auch sonst nichts, was nicht gut in das AppEngine Modell passen würde. Das vermutlich schwierigste ist das Ansprechen von Druckern von der AppEngine aus. 

Aber ... Paketaufkleber haben in der Regel relativ viele Nummern und Barcode. Einer davon ist üblicherweise die "Paketscheinnummer". Die dienet auch der Abrechung mit dem Paketdienstleister ("KEP", wie wir Logistiker sagen: *K*urrier, *P*aket, *E*xpress). Wir kriegen vom Diensteister eine viertel Million Nummern zugeteilt (z.b. 4901214213405250000 bis 4901214213405500000), die wir dann Munter als Paketscheinnummern verwenden. Nach ein paar Monaten sind die aus und wir bekommen einen anderen Nummernblock zugeteilt.

Das liess isch in PostgreSQL ganz gut mit `Sequence`s regeln. Zwar hatten wir jedes mal den [`CREATE SEQUENCE`][2] Syntax vergessen, aber im grossen und ganzen lief es. Es gab irgendwo in der Schublade Pläne, den Wechsel von einem zum anderen Nummernkreis zu automatisieren, aber es gab immer andere Projekte, die dringender schienen.

Leider gehört es zu den Gorssen Problemen in verteilten Systemen, einen Zähler, beziehungsweise eine Sequenz hochzuzählen. Das merkt man schon bei relativ simplen Ansätzen, wie [Multi-Master Replikation][3] oder [Sharding][4] - die Primärschlüssel, die oft als Sequenz/Counter realisiert  sind, können dort erhebliche Kopfschmerzen bereiten.

In der Literatur zu verteilten Systemen landet man schnell bei [Vektoruhren][5] und der gleichen. Alles nciht mal eben zu implementieren - insbsondere auf der AppEngine, wo ich keinerlei Kontrolle habe, auf welchen Knoten meine Applikation läuft.

[Nachfrage auf Stackoverflow][6] brachte einige interessante spekte zum Vorschein. Mit folgendem Code kriegt man recht brauchbar einigermassen monoton steigende Zahlen - insbesondere wenn man `Model` für cnihts anderes benutzt:

    handmade_key = db.Key.from_path('Model', 1)
    counter = db.allocate_ids(handmade_key, 1)[0]

Damit ist aber nicht sichergestellt, das es in der sequenz keine Lücken gibt. Wenn man das wirklich braucht, muss man mit einem [Singelton][7] in der Datenbank arbeiten, dass man innerhalb einer Transaktion updatet. Das kann das beispielsweise so aussehen:

    def _get_numbers_helper():
        seq =  Sequence.all().get()
        seq.counter += 1
        seq.put()
        return seq.counter-1
    
    def get_numbers(needed):
        return db.run_in_transaction(_get_numbers_helper)

Weil man eine Transaktion benutzt, ist dsa ganze aber recht langsam. Man muss fuer eine TRansaktion 100-200 ms rechnen und durch optimistick locking kann es bei vielen gleichzeitigen Versuchen, Sezenznummern hochzuzählen ganz schnell dazu kommen, dass der Durchsatz weit hinter dme gewünschten zurückbleibt. Für unserem Bedarf der Paketscheinnummern scheint es aber zu reichen.

Im [Google AppEngine Toolkit (gaetk)][8] gibt es das Modul [`gaetk/sequences`][9] das die Sequenzgenerierung auch mit mehreren Nummernblöcken formschön und praktisch abbildet.


[1]: http://blogs.23.nu/disLEXia/tag/hulog/
[2]: http://www.postgresql.org/docs/8.1/static/sql-altersequence.html
[3]: http://en.wikipedia.org/wiki/Multi-master_replication
[4]: http://en.wikipedia.org/wiki/Shard_(database_architecture)
[5]: http://de.wikipedia.org/wiki/Vektoruhr
[6]: http://stackoverflow.com/questions/3985812
[7]: http://de.wikipedia.org/wiki/Singleton_(Entwurfsmuster)
[8]: https://github.com/mdornseif/appengine-toolkit/#readme
[9]: https://github.com/mdornseif/appengine-toolkit/blob/master/gaetk/sequences.py
