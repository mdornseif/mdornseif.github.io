---
layout: post
title: Appengine Authentifizierung I
---

{{ page.title }}
================

Moderne Authentifizierung sollte den Nutzern möglichst wenig Wissen um Accounts und Passwörter abverlangen. Es ist nicht nur unsicher, wenn Nutzer für verschiedene interne Applikationen verschiedene Nutzernamne und Passwörter wissen müssen, sondern auch unakzeptabel. Spuatestens bei einem Duzend Applikationen steigt keiner mehr durch, welche Zugangsdaten zu welchem System gehören und entweder wird überall das Gleiche Passwort verwendet oder es werden in jedes System solange alle Passwörter eingegeben, bis man "drin" ist.

Authentifizierung für AppEngine Applikationen ist mit den [`google.appengine.api.users` API][1] ungemein
einfach zu implementieren, wenn man verlangt, dass jeder Nutzer einen Google Account hat. Alternativ kann der, der "Google Apps for your domain" nutzt, [gegen diese Accounts authentifizieren][2]. Seit kurzer Zeit unterstützt AppEngine auch [Federated Login][3] mit [OpenID][4]. Dass dumme ist, dass man diese Optionen nicht so ohne weiteres zusammen nutzen kann. Oh, und wenn man "Google Apps for your domain" als Authentifizierung gewählt hat, gibt es keine Möglichkeit, das nachträglich zu ändern.

[1]: http://code.google.com/appengine/docs/python/users/
[2]: http://code.google.com/appengine/articles/auth.html
[3]: http://code.google.com/appengine/articles/openid.html
[4]: http://de.wikipedia.org/wiki/Openid


Authentifizierung gegen mehr als eine "Google Apps for your domain"
-------------------------------------------------------------------

Wir arbeiten auf einigen unserer selbst entwickelten Applikationen zu mehr als einem Unternehmen zusammen. Deswegen wollen wir die Benutzer auch gegen verschiedene Google Apps Domains Authentifizieren - ohne unsere Applikation dauerhaft auf "Restricted to the Following Google Apps Domain" umzustellen.

Die Lösung hierfür ist OpenID. "Google Apps for your domain" kann als OpenID Provider dienen - wenn auch mit einigen Macken. Eine Google AppEngine Applikation kann ein OpenID "Consumer" sein und die zugehörigen Libraries unterstützen sogar die "seltsamme" OpenID Implementierung von Google Apps.

Wenn man für seine Applikation im AppEngine Dashboard unter "Application Settings" die "Authentication Options" auf "(Experimental) Federated Login" gestellt hat ist _fast_ alles erledigt. Google kümmert sich um Session Coockies und dergleichen. `google.appengine.api.users.get_current_user()` liefert den aktuell eingeloggten user zurück. Wir müssen nur noch dafür sorgen, das User eingeloggt werden - und zwar nur von den "Google Apps for your domain" Domains, von denen unsere Nutzer kommen.

Dazu wird von der AppEngine erwartet, dass man unter der URL `/_ah/login_required` code laufen hat, der sich um den Login kümmert. Nicht eingeloggte user sollte man daher dahin umleiten. Das lässt sich mit Googles Webapp Framework z.B. in ein Mixin packen:

<script src="http://gist.github.com/630972.js?file=openid_mixin.py"></script>

Wenn der Nutzer eingeloggt ist, wird sein Nutzerobjekt zurückgegeben, ansonsten wird der Nutzer auf `/_ah/login_required` umgeletiet. Im Parameter `continue` wird übergeben, wohin der Nutzer nach erfolgreicher Authentifizierung hin zurück geschickt werden soll.

Ein Handler, der Authentifizierung wünscht, muss nun einfach `self.get_user()` aufrufen. Wenn `None` zurückgegeben wird, war die Authentifizierung erfolgreich. Der handler sollte dann return aufrufen - durch `get_user()` wurden schon alle Header entsprechend gesetzt, um einen redirect herbeizuführen. Wenn `get_user()` ein User-Objekt zurückgegeben wird, sit das der aktuel authentifizierte Nutzer für den Inhalte gerendert werden können. Ein entsprechender Handler könnte z.B. so aussehen:

<script src="http://gist.github.com/630972.js?file=main.py"></script>

Jetzt fehlt uns noch der Handler für `/_ah/login_required`. Dieser muss die "Google Apps for your domain" Domain ermitteln, gegen die der Nutzer authentifizieren möchte, dann die entsprechende OpenID Login URL konstruieren und den Nutzer dahin umleiten. Eine entsprechende Implementierung sieht so aus:

<script src="http://gist.github.com/630972.js?file=do_openid_login.py"></script>

Wenn der Benutzer frisch auf `/_ah/login_required` gelangt, wird der Code nach `if not openid_url:` ausgeführt: Ihm wird [dieses HTML-Template](http://gist.github.com/630972#file_login.html) verwendet. Es stellt die Logos der Zwei Domains (example.com & company.local), mit denen man sich einloggen kann, dar. PEr Klick kann man eine auswählen und dadurch wird wieder der obige Handler wieder aufgerufen. Diesmal sollte durch die GRafischen Buttons `example.com.x` oder `company.local.x` gesetzt sein und darauf basierend wird die zu verwendene `openid_url` gesetzt. Hier könnte man Problemlos auch andere OpenID Provider anflanschen.

Mit `self.redirect(users.create_login_url(continue_url, None, openid_url))` wird der User in die Google OpenID Maschinerie weitergeleitet, die ihn nach erfolgreicher Authentifizierung zu der `continue_url` schicken sollte.

Damit haben wir eine saubere und einfache Authentifizierung für unsere Mitarbeiter - es ist sogar nur sehr wenig Code dafür nötig.


Basic-Auth fürs API
-------------------

Das alles ist schön und gut, aber wir brauchen auch Authentifizierung für API Zugriffe, und die soll per [HTTP Basic Auth][5] gehen. Warum kein [OAuth][6] ([RfC 5849](http://tools.ietf.org/html/rfc5849))? OAuth ist eine feine Sache, aber fordert den Client-Entwicklern deutlich mehr ab, als Basic-Auth, die heutzutage von verst jedem HTTP-Client unterstuutzt wird. Die gruoßten Sciherheitsprobleme von Basic-Auth lassen sich durch SSL/TLS lösen und dadurch, dass nur systemgenerierte User:Passwort Kombinationen verwendet werden. Um die [OAuth Webseite][7] zu zitieren: "OAuth 1.0 is still not trivial to use on the client side. The convenient and ease offered by simply using passwords is sorely missing in OAuth. For example, developers can quickly write Twitter scripts to do useful things by using their username and password. With the move to OAuth, these developers are now forced to find, install, and configure libraries in order to accomplish what was before possible with a single line of cURL script."

Also bleiben wir ganz klassisch bei Usernamen udn PAsswort, oder `uid` und `secret` - beide Werte haben nämlich mit dem User, der sie nutzt wenig zu tun. Wir müssen diese zunächst in der Datenbank abbilden können.

<script src="http://gist.github.com/631198.js?file=models.py"></script>

Mit `Credential.create()` krieg ich so  `uid` und `secret`. Jetzt muss ich die HTTP-Authentifizierung prüfen. Anstatt wie oben mit einem Mix-In zu arbeiten überschreiben wir diesmal die `initialize()` Methode der `webapp.RequestHandler`-Klasse. Viele andere Web-Frameworks bieten hier deutlich elegantere Methoden in den Request/Response Mechanismus einzugreifen, aber das bei der AppEngine mitgelieferte `webapp` Toolkit ist an dieser Stelle sehr spartanisch.

<script src="http://gist.github.com/631198.js?file=MyHandler1.py"></script>

Der Code ist unaufregend: Wenn die Userdaten stimmen, wird `self.credential` gesetzt, ansonsten der HTTP Status Code 401 zurückgegeben.


Authentifizierung über Sessions mit eigener Nutzerdatenbank
-----------------------------------------------------------

Wenn wir neben OpenID auch mit einer eigenen Nutzerdatenbank arbeiten wollen, müssen wir zuerst unser eigenens Session System implementieren. Wir nutzen für die Sessions gaesessions[8]. Dafür müssen wir prüfen, ob bereis eine Session vorhanden ist und wenn nciht den Nutzer auf die altbekannte Login URL umleiten.

<script src="http://gist.github.com/631198.js?file=MyHandler2.py"></script>

Der Code für `/_ah/login_required` muss nun mit einer einfachen Username und Passwort Eingabe umgehen können und prüfen, ob entsprechende Credentials in der Datenbank existieren. Obendrein muss natürlich bei Bedarf ein Login-Formular dagestellt werden, wo Uid und Secret eingegeben werden können.


<script src="http://gist.github.com/631198.js?file=do_openid_login1.py"></script>
