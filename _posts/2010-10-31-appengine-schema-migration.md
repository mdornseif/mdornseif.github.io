---
layout: post
title: Schema Migration auf der AppEngine
---

{{ page.title }}
================

Das Google AppEngine Datastore ist dem Grunde nach "schemaless". Dass bedeutet, das jede Datenbank-Spalte
unterschiedliche Zeilen haben kann. Das haben Entwickler nicht so gerne, deshalb basteln sie sich in aller
Regel ein Schema auf das Datastore. Z.B. so:

    class Rechnungsliste(db.Model):
        kundennr = db.StringProperty(required=False)
        monat = db.DateProperty(required=True)
        rechnungen = db.ListProperty(db.Key, default=[])

Im einfachten Fall kann man einfach eine weitere Spalte ("Property", wie es bei Google genannt wird) zufügen.
Bei Datenbankzeilen, die noch nach dem alten Schema geschrieben wurden, hat diese Spalte dann den Wert
`None`. Wenn der eigene Code damit klar kommt ist die migration damit abgeschlossen.

    class Rechnungsliste(db.Model):
        kundennr = db.StringProperty(required=False)
        monat = db.DateProperty(required=True)
        rechnungen = db.ListProperty(db.Key, default=[])
        processed_at = db.DateTimeProperty()

Bei einem Boolean Property ist das in der Regel kein Problem, dem im Python Altagsgebrauch sind `None` und
False eh das selbe. Man kann sich aber dann doch auch erheblich ins Bein schiessen, insbesondere, wenn man
mit `required=True` arbeitet. Nehmen wir folgendes Beispiel:

    class Rechnungsliste(db.Model):
        kundennr = db.StringProperty(required=False)
        monat = db.DateProperty(required=True)
        rechnungen = db.ListProperty(db.Key, default=[])
        lieferantennr = db.StringProperty(required=True, default='')

Wenn man das nächste mal versucht, eine "alte" Rechungsliste zu laden, wird man mit einem `BadValueError:
Property lieferantennr is required` bestraft. Wir müssen also alle vorhandenen Datensätze anpassen. Dazu
nutzt man am besten [appengine-mapreduce](http://code.google.com/p/appengine-mapreduce/), die ins
rootverzeichnis des Projektes entpackt wird und in der `app.yaml` als handler eingebundenw ird:

    handlers:
    - url: /mapreduce(/.*)?
      script: mapreduce/main.py
      login: admin

Damit können wir jetzt die lieferrantennummer jeder Rechungsliste anpassen. Dazu nehmen wir erstmal wieder
`required=True, default=''` aus der Model-Definition und sagen in `mapreduce.yaml`, dass wir bitte über jedes
Objekt vom Typ `models.Rechnungsliste` iterieren wollen.

    mapreduce:
    - name: rechnungslistenupdate
      mapper:
        input_reader: mapreduce.input_readers.DatastoreInputReader
        handler: mapping.rechnungslistenupdate
        params:
        - name: entity_kind
          default: models.Rechnungsliste

In `mapreduce.py` (oder auch in jeder anderen Datei) ist jetzt die funktion definiert, die mit jeder
einzelnen Rechungsliste als PArameter aufgerufen wird.

    from mapreduce import operation as op
    
    def rechnungslistenupdate(rechnungsliste):
        if not rechnungsliste.lieferantennr:
            rechnungsliste.lieferantennr = ''
        yield op.db.Put(rechnungsliste)

Etwas ungewohnt mag sein, dass wir nicht `rechungsliste.put()` aufrufen, sondern `op.db.Put(rechnungsliste)`
zurücukliefern. Das ermögtlich es der mapreduce library mehrere Schreibvorgänge zu einem Batch
zusammenzufassen und damit erheblich effizienter zu arbeiten.

![MapReduce Status Screen](http://static.23.nu/md/Pictures/ZZ56D86E42.png)

Das war's schon. Jetzt kann mn auf http://localhost:8000/mapreduce gehen und den Job `rechnungslistenupdate` starten. Je nach Datenmenge sollte die Bearbeitung einige Augenblicke bis einige Stunden dauern. Die Library unterteilt die Aufgabe in "Shards" und lässt diese jeweils über einen Deil der Daten laufen. 100 Datensätze pro Sekunde und 8 Shards sind die Default Parameter, die auf der "echten" AppEngine in Produktion auch gut Erzeiht werden können - damit ist man dann bei 800 Datensätzen, die man pro sekunde Updaten kann - gar nciht so schlecht.

![MapReduce Job Display](http://static.23.nu/md/Pictures/ZZ13EFCF04.png)

Das haupt problem momentan ist, das es praktisch kein Error Reporting gibt und huangende Jobs nur durch manuelles bearbeiten des Datastores abgebrochen werden können. Also: Code besser vor der Benutzung zwei mal durchlesen!