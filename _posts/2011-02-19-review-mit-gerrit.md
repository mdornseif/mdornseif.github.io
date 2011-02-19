---
layout: post
title: Wir wir entwickeln - Code Reviews mit Gerrit
---

{{ page.title }}
================


Im letzten Jahr hat sich bei uns im Unternehmen die Software-Entwicklung dramatisch verändert: Neue Entwickler. Unser Werksstudent ist (endlich) Vollzeit dabei. Inzwieschen haben wir mit 4 fähigen Software Entwickler Vollzeit beschäftigt. Gleichzeitig haben wir es geschafft, einige alte Zöpfe bei unserer Software abzuschneiden: Systeme bei denen nie so ganz klar war, ob dafür die Entwickler (“Dev”) oder die Administratoren (“OPs”) zuständig waren, sind abgeschafft oder deutlich zurück gefahren worden. Aufwändig zu betreibende Infrastruktur für unsere selbstgeschreibene Software (wie PostgreSQL, Memcache, MySQL, RabbitMQ, Redis, Fileservices, der Django+WSGI+Apache Stack, LDAP, Mailserver usw.) wird immer weniger. Cloud Services machen Feployment und Installation immer einfacher. Inzwischen haben ein Grossteil unserer Applikationen ein `make deploy` target. Wir sind drei sehr grosse und nervrnaufreibende Softwarepakete los geworden. Eigentlich haben wir also viel mehr Zeit, um neue, grossartige Software zu entwickeln. Und genau das haben wir uns auch vorgenommen. Und auch gemacht. Oft auch mit Erfolg.

![Code-Tree](http://static.23.nu/md/Pictures/huLogi.png)

Lange Zeit wurde der Code, den wir einsetzten überwiegend von den beiden Dornseifs außerhalb der Arbeitszeit geschrieben, dann zusätzlich von einem Entwickler fulltime mit gelegentlichen Einsprengseln eines Werksstudenten. Da bekam man praktisch automatisch mit, was der andere so programmierte. Mit deraktuellen Teamgröße ist der vorbei. Das bereitet Probleme.

Eine Zeitlang konnten wir die Probleme dadurch auffangen, dass wir mächtige neue Werkzeuge einsetzten, wie verteilte Versionskontrolle (GitHub mit allen Features). Das führte dazu, dass es nicht so tragisch war, wenn der eine nicht ganz mitbekam, was der andere so trieb. Eine Zeitlang.

Aber alles in allem ist unsere Entwicklung auch immer wieder durcheinander gegangen und der Code erreichte oft nicht die Qualität, wie sie von den anderen Abteilungen im Unternehmen erwartet wird. Auch haben wir inzwischen jede Menge von nie zu Ende programmierten Branches rumliegen.

![Code-Tree](http://static.23.nu/md/Pictures/EDIhub3.png)

Die in diesem Artikel eingestreuten Bilder zeigen alle verschiedene Bereiche unserer internen Software. Jede Linie ist ein eigener Entwicklungszweig. Ich denke, es ist für jeden deutlich zu sehen, dass das oft ein wildes hin- und her ist. Dabei den Überblick zu behalten ist extrem schwierig.

Ein Problem mag sein, das wir von den Möglichkeiten der verteilte Versionskontrolle vielleicht etwas zu enthusiastisch gebrauch gemacht haben. So hat es vom EDIhub in den letzten 3 Tagen 10 (!) unabhängige Versionen/Entwicklungszweige ("Branches"") gegeben. Sowas kann nicht gut gehen.

Die Gründe für all die Probleme sind viellfältig, aber wie so oft stinkt der Fisch vom Kopf. Ich bin bekennender [Cowboy Coder](http://cowboyprogramming.com/2007/01/11/delving-into-cowboy-programming/) und halte viele Planungen, die bei Software-Projekten laufen für eine Ausrede um nicht implemnetieren zu müssen - ind einen Großteil der Testprogrammiererei in vielen Fällen für eine Zeichen mangelnden Selbstvertrauens (vielleicht zu recht!). 

Die Methode klappt, führt zu hohher Produktivität, gutem Code, robuster, felxibler Software und klappt bei
einem sehr kleinen, sehr gut zusammenarbeitenden Team auch ganz gut. Aber jetzt klappt sie bei uns nicht mehr gut genut.

![Code-Tree](http://static.23.nu/md/Pictures/huLogi.png)

Der Lösungsansatz deen wir gewählt haben ist 

1. weniger, aber größere Änderungen (“Commits”)
2. weniger auseinander laufende Entwicklung (“Branches”)
3. jedes Stück Programmcode sollte durch die Hände von zwei oder drei Entwicklern gehen (“Code Reviews”)

Das wird alles in allem zu einer Entschleunigung unserer Software-Entwicklung führen. Details zur Durchführung im Folgenden.

![Code-Tree](http://static.23.nu/md/Pictures/pyJasper.png)


### Reviews

Ein Code Review als Teil eines Software-Entwicklungsprozesses legt fest, dass jedes Stück neu entwickelter
oder geänderter Programmcode von einem Kollegen durchgeschaut wird. Wir hatten vor knapp 2 Jahren schonmal
einen Review Prozess (mit Howsmycode, Reviewboard und [Google Code
Reviews](http://www.google.com/enterprise/marketplace/viewListing?productListingId=5143210+12982233047309328439)
begonnen, aber so recht hat das nie geklappt. Seitdem hat sich viel geändert: wir sind doppelt so viele
Entwickler, wir verwenden ein anderes Versionsverwaltungssystem (git statt Subversion), wir haben einen
Continuous Integration Server
([Jenkins](http://blog.codecentric.de/2011/02/jenkins-continuous-integration-reloaded/)), der unsere Software
nach jeder Änderung testet.

Also starten wir einen neuen Versuch. Ziel ist, dass Programmcode, bevor er in Betrieb geht, immer zu prüfen und immer von Kollegen durchsehen zu lassen. Der Artikel [Someday, all software will be built this way](http://alblue.bandlem.com/2011/02/someday.html) erklärt die Vision dahinter.

Wir nutzen ein Programm Namens [Gerrit](http://code.google.com/p/gerrit/) um die Software Quaitätskontrolle zu steuern. Gerrit sitzt zwischen der Quellcodeverwaltung (“Repository”) des einzelnen Entwicklers und dem zentralen Repository, dass wir bei der Firma GitHub betreiben. Als Entwickler schicke ich meinen Code nicht mehr direkt zu GitHub, sondern zu Gerrit. Dort kann der Programmcode von den Kollegen begutachtet werden. Eventuelle Verbesserungen können vorgeschlagen, implementiert und erneut zu Gerrit geschickt werden.

Wenn der Code von den Kollegen als hinreichend gut angesehen wird, wird er genehmigt und von Gerrit an Github weiter gesendet. Der einzelne entwickler kann gar nciht mehr direkt Code in unsere Zentrale Quellcode-Verwaltung (GitHub) schreiben. Gerrit ist der "Gatekeeper", der sicherstellt, das aller Code revied wird. Die Installation von Software auf unseren Servern erfolgt imemr aus Github heraus. Damit ist sichergestellt, das nur reviewter Code zum Einsatz kommt.

![Code-Tree](http://static.23.nu/md/Pictures/EDIbub2.png)


**Was haben die Reviewer nun zu tun?** Zwei Sachen: erstens “Verification” und zweitens “Review”.

Verification ist die Überprüfung, ob Code unseren formalen Anforderungen genügt oder ob grundlegende Fehler, wie Syntax Fehler vorliegen. Ein Auschecken mit nachträglichem `make check` Test stellt eine Verification dar. Auf Dauer ist das sicher etwas, was man mit Hudson voll automatisieren kann, ansonsten wird das vom ersten Reviewer erledigt, der den Code unter die Finger bekommt.

` mcke check` Testet unseren Code vor allem mit [pyflakes](http://pypi.python.org/pypi/pyflakes) und [pep8.py](https://github.com/cburroughs/pep8.py). Das hat frei Vorteile:

1. Findet es gar nicht so selten Syntax fehler
2. Findet es gelegentlich vergessene imports
3. Zwingt es uns zu einem enheitlichen Coding Style.

Der dritte Punkt ist im Team ehr unbeliebt. Das gemeckere über "hier eine Leerzeile zu viel", "dort ein
ungenutzter import" oder "hier ein Leerzeichen zu wenig" nervt schon. Und es sind ja alles nciht wirklich
fehler, selbst wenn unsere [Coding
Guidelines](https://github.com/hudora/huTools/blob/master/doc/development.markdown) PEP8 vorschreiben. Stimmt. Aber es führt zu einem einheilichen Stiel alles unseres codes. Und dass macht meienr ERfahrung nach unsere Changesets sehr viel lesbarer. Bevor wir PEP8 checks eingeführt hatten, waren in jedem Changeset auch immer eirgendwelche Whitespace fixes, die "im vorbeigehen" erledigt wurden. Das machte die Sache recht unübersichtlich. Jetzt sind solche Fragen in der gesammten Codebasis einheitlich gelösst und die Changesets deutlich übersichtlicher.

![Code-Tree](http://static.23.nu/md/Pictures/produktpass.png)


Der Review selbst soll Mängel im Produkt entdecken. Der Reviewer studiert dazu den vom Autor gelieferten Code und im Kontext der bereits vorhandenen Codebasis und der Anforderungen, die dieser Code implementieren soll. Der Reviewer muss dabei sowohl Implementierungsdetails auf Fehler hin untersuchen, als auch das Design, auf dem die Implementierung basiert. Er muss auch prüfen, ob der Code das angestrebte Zeil (in der REgel in einem Ticket beschreiben) angemessen implementiert und ob das Ziel überhaupt implementierbar sit.

Der Reviewer muss den Code und die Testcases nicht ausführen. Seine Aufgabe ist es nicht, Fehler zu entlarven, die automatisiert gefunden werden können. Wo eine GUI vorhanden ist sollte sich der Reviewer diese anschauen. Dazu solten im Commit Screenshots referenziert sein.

Der Reviewer schaut im Einzelnen sich die Änderungen unter folgenden Gesichtspunkten an:

1. Lesen der Change Notes. Sicherstellen, dass die Änderungen der Aufgabe / dem Ticket angemessen sind, und dass verständlich beschrieben wird, was durch die Änderung Implementiert wird. Für GUI-Bezogene Änderungen sollten die Change Notes entsprechende Screenshots beinhalten.
2 Lesen der Änderungen der Dokumentation. Existiert Dokumentation außerhalb (README o.ä) und innerhalb (Module-Docstrings, Kommentare) des Codes, die geänderte und neu implementierte Funktionalität hinreichend beschreibt? Ist verständlich welche Funktionalität nach der Änderung vorhanden ist?
3. Überprüfen, ob der Code verständlich ist. Versteht der Reviewer ohne Anstrengung, was der Code macht?
4. Überprüfen, ob geliefert wird, was versprochen wurde. Sicherstellen, dass der Code die in den Change Notes und der Dokumentation genannte Funktionalität implementiert. Und keine Fehler - insbesondere Logischer Art beinhaltet
5. Überprüfen, ob der Code kompatibel ist. Sicherstellen, das der Code mit der restlichen Architektur im Unternehmen harmoniert. Dazu gehören [Coding](https://github.com/hudora/huTools/tree/master/doc)- und [Benennungsstandards](https://github.com/hudora/huTools/tree/master/doc/standards) (AddressProtocol etc.), aber auch Nutzung bereits vorhandener Library Funktionen. Auch die Einhaltung von Unternehmensgrundsätzen, wie der nicht-Maskierung von Fehlern und dem “Fail fast, fail early” Prinzip sind zu prüfen.

Prüfen der Teststruktur. Sicherstellen, dass der Code dort, wo er mit vertretbarem Aufwand testbar ist, auch getestet werden kann. Der Reviewer sollte dabei konkrete Hinweise geben, wie der Code getestet werden kann - möglicherweise ist der Code gegen Interessante Fehler (z.B. Timeouts) nicht sinnvoll testbar.

[Diese Erklärung](http://review.webmproject.org/Documentation/user-signedoffby.html#Reviewed-by) gibt noch einmal gut wider, was der Reviere mit seinem Review bestätigt.

![Code-Tree](http://static.23.nu/md/Pictures/CentralServices.png)


### Weniger Commits

Der einzelne Entwickler kann auf seinem eigenen Rechner und seinen eignen GitHub Repositories nach Lust und Laune branchen und committen. Bei langwierigeren Projekten oder zum Datenaustausch mit Kollegen, wird empfohlen, auf GitHub einen “Fork” zu machen, wo der Code gelagert werden kann. Wenn alles fertig programmiert ist, dann werden all Commits, die der Entwickler gemacht hat, zu einem einzigen zusammengefasst (git rebase/squash) und in das Review System geladen. Änderungen aufgrund des Reviews werden auch wieder mit der Original-Änderung zusammengefasst (squash!) und ans Review System geschickt, bis die Reviewer zufrieden sind. Dann wandert dieser eine Commit in den “master” auf GitHub.

Die Menge Code, die jeweils in einem Review landet sollte irgendwo zwischen 20 und 200 Zeilen - zur Not auch in bisschen mehr - liegen.

Das ganze ist recht ungewohnt für uns denn bisher war ein einmal gemachter Commit eigentlich in unserem Workflow "immunable" und wurde ncihtmehr geändert. Mit den Reviews wird es jezt dazu kommen, das ein Changeset vielleicht ein halbes Dutzend mal überarbeitet werden muss, bis es als ein einziger Commit im Repositort landet.


![Code-Tree](http://static.23.nu/md/Pictures/huSoftM.png)


### Weniger Branches

Bisher haben wir in der REgel PRo Ticket einen branch in gitHub angelegt. Das wird sich radikal ändern.
Je nach Projekt arbeiten wir nur noch mit einem Branch (master) oder mit einem aktuellen Branch (master) und dem Branch für das nächste größere Rollout (zB edihub_v3). Bei Libraries werden wir in der Regel nur einen Branch haben, bei “grossen” Webapplikationen zwei und bei kleinen Webapplikationen schau’n wir mal.

Branches können nur über die Admin-Oberflähce des Review-Tools angelegt werden und sollten nur nach Rücksparache angelegt werden. Wie bereits gesagt, in seinem eigenen Repository kann und soll jeder Entwickler nach Lust und Laune branchen, wir übernehmen diese Branches aber nicht mehr in unser Haupt-Repository - da darf der einzelne Entwickler gar nicht mehr schreibend drauf zugreifen.

Projekte nutzen immer den master Branch unserer Libraries. Das bedeutet, um ein Feture im Web-Frontentd zu implmentieren, dass eine neue Library Funktion benötigt, muss ich erst die Library-Funktion reviewen lassen und in den Master bekommen und kann erst danach die Funktionalität im Web-Frondend Revieren lassen.

Das wird einige Entwicklungsprozesse bei uns deutlich entschleunigen.

![Code-Tree](http://static.23.nu/md/Pictures/DeadTrees2.png)

## Implementierung

Wir beginnen damit Randbereiche unserer Software auf das neue System umzustellen. Zunächst huBarcode, pyJasper und gaetk (appengine-toolkit). Danach ziehen wir stück f¨ru Stück die grossen Projekte u,
Gerrit ist ein vollständiger Git server, so dass man bei Bedarf von dort aus seinen Code clonen kann:

    git clone ssh://mdornseif@cybernetics.hudora.biz:29418/huBarcode

Man kann sein Reopository aber auch weiterhin von GitHub clonen, sollte GitHub aber in Zukunft als Read-Only ansehen. Das lLben einfacher macht man sich, wenn man sich per `scp -p -P 29418 mdornseif@cybernetics.hudora.biz:hooks/commit-msg .git/hooks/` noch die Commit Messages umformatieren lässt. Nachdem man nun munter vor sich hin programmiert hat, commitet man und pusht die Änderung zu Gerrit. Aber nicht wie üblich (und mit default-einstellungen einfach gemacht) nach refs/heads/master sondern nach refs/for/master, also

    git push ssh://mdornseif@cybernetics.hudora.biz:29418/huBarcode HEAD:refs/for/master

Das war’s schon, in Gerrit sollte jetzt ein review-Request, wie zB http://cybernetics.hudora.biz/gerrit/#change,1 gelandet sein.

Die Reviewer können sich jetzt leicht den Branch mit den Änderungen ziehen. Auf den Geriet Review Seiten gibt es entsprechende Cut’n Paste Beispiele. Z.B.

    git fetch ssh://mdornseif@cybernetics.hudora.biz:29418/huBarcode refs/changes/01/1/4
    git checkout FETCH_HEAD

Wenn er Code wegen des Reviews geändert werden soll, macht man das, comitted und “squashed” den neuen commit dann mit dem vorherigen zusammen:

    git rebase -i HEAD~2

Dabei sollte man auf jeden Fall die obere `Change-Id: I...` Zeile intakt lassen und die untere entfernen. Sonst kann Gerrit nicht zuverlässig erkennen, wo der Commit hin gehört. Wenn man nun erneut `git push gerrit HEAD:refs/for/master` macht, wird die neue Änderung anhand von `Change-Id` zugeordnen. Alternativ könnte man direkt nach `refs/changes/1` (die letzten beiden Pfadelemete weg und führende Nullen entfernen) pushen, dann würde die Änderung auf jeden Fall beim richtigen Review landen. Alternativ kann man wohl `git commit --amend` verweenden, das hab ich mir aber noch nicht angeschaut.

![Code-Tree](http://static.23.nu/md/Pictures/gaetk.png)

### Wann und wie?

Reviews sollten immer zeitnah erledigt werden. Insbesondere sollte man keine neuen Tickets oder sonstige Projekte beginnen, wenn man stattdessen noch Reviewen arbeiten könnte. Der Reviewer ist nicht da, um den Reviewten zu hänseln, sondern um a) selber besser den Code zu verstehen und b) den Code besser zu machen. Es geht dabei nicht um die Person des Autors. Es ist völlig akzeptae;, im Review die Meinung zu äussern, dass Code so nicht in Produktion gehen sollte (-1). Es ist auch ok, wenn andere Reviewer den Reviewer überstimmen.

Der letzte Reviewer, mit dem der Code als “gut” befunden wird, merged dann mittels des “Submit Patch Set” Button. Gerrit befördert den Code dann selbständig auf GitHub.



Mehr zum Thema gibt es in der [Gerrit Dokumentation](http://gerrit.googlecode.com/svn/documentation/2.1.6/user-upload.html#), [Working with Gerrit](http://avogadro.openmolecules.net/wiki/Working_with_Gerrit), [Code Review with Gerrit, a mostly visual guide](http://unethicalblogger.com/node/241) (wohl ein [ehr untypischer Workflow](http://groups.google.com/group/repo-discuss/msg/d43ba0bb0b74daca)), [Hacker news: unhappy with gerrit](http://apps.ycombinator.com/item?id=2206785), [Gerrit: Google-style code review meets git](http://lwn.net/Articles/359489/), [Submitting to WebM](http://www.webmproject.org/code/contribute/submitting-patches/), [EGit Contributer Guide](http://wiki.eclipse.org/EGit/Contributor_Guide#Gerrit_Code_Review_Cheatsheet), [Wikibook Gerrit Code Review](http://en.wikibooks.org/wiki/Git/Gerrit_Code_Review).
