---
layout: post
---

# Eine Depotverwaltung mit React, AG-Grid, GraphQL, Express und SQLite

Banken und Broker versuchen ja Börsenhandel durch Gamification mit möglichst viel Suchtpotentia zu versehen. Vor allem zeigen sie einem nicht grade das an, was einen davon abhalten könnte noch mehr zu zocken, sondern bringen orimär erfolgsmeldungen nach vorne.

So wird üblicherweise der Gewinn und Verlust durch Verkauf gar nciht kummuliert angezeigt. Auch kann man oft Depot und Orderbook nicht gemeinsam anzeigen. Und wenn es um mehrere Broker geht ist praktisch sofort Schluß. Dazu kommt, dass es kaum vernünftige APIs und Export-Möglichkeiten gibt.

Da muss man also mit eigener Software-Entwicklung ran. Extraktion von Informationen aus obscuren Exports, PDFs und HTML-Scraping. Verwaltung des ganzen in SQLite - wir wollen ja keine Multi-User App bauen, also kein Bedarf für eine größere, langsamere Datenbank - und Frontend mit React und da als Star des ganzen ag-Grid zur Darstellung der verschiedensten Tabellen und Grafiken.

