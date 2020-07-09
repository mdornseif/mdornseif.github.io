# Code Formatierung und Upgradibility mit pre-commity erzwingen in der Praxis

Wenn man mit mehreren Entwicklern an einem Projekt arbeitet, ist es sehr hilfreich sich auf einen /coding style/ zu einigen. Nicht nur aus ästhetischen Gründen, sondern auch, weil dadurch merges erheblich robuster werden.

Am besten ist es natürlich, sich nicht auf vage Richtlinien zu verlassen, sondern das ganz als code festzulegen.

Mit dem Toolkit [pre-commit](https://pre-commit.com) (verwirrenderweise genau wie der Git-hook benannt) kann man sicherstellen, das alle bei einem commit geänderten Dateien den Style-Guidelines entsprechen, die man sich so gesetzt hat. 

Das hilft auch ganz gut, eine große Codebase langsam zu migrieren: nur geänderte Dateien muss man (komplett) auf die neuen Style-Guidelines anpassen. Und wenn man nur einen winzigen Bugfix macht, und nicht die ganze Datei updaten will, kann man das ganze mit `git commit -n` auch mal ausschalten.

Ein gutes Vorgehen ist es, das die CI-Scripts automatisiert die gröbsten Schnitzer bezüglich der Style-Guidelines erwischen und verhindern und das man in den pre-commit Hooks sehr viel strenger ist, die kann man ja im Einzelfall umgehen.

Hier die Beispiel [pre-commit Hook](https://pre-commit.com/hooks.html) Konfiguration für ein Python 2.7. Projekt auf der Google App Engine.

## pyupgrade

Zunächst führen wir [pyupgrade](https://github.com/asottile/pyupgrade) aus, um  veraltete Sprachkonstrukte automatisch zu modernisieren und auf Python 3.x vorzubereiten:

    - repo: https://github.com/asottile/pyupgrade
      rev: v1.11.3
      hooks:
        - id: pyupgrade

## black

Als nächstes wird der Sourcecode mit [black](https://github.com/psf/black) formatiert:

    - repo: https://github.com/ambv/black
      rev: stable
      hooks:
        - id: black
          language_version: python3.7
          args: ['--skip-string-normalization']

black ist nicht perfekt, sorgt aber sehr zuverlässig dafür, das alle commits Whitespace, Klammern, etc an der richtigen Stelle haben.  Da black Python 3 benötigt, stellen wir sicher, das pre-commit das Programm mit der richtigen Python Version ausführt. Also: black läuft auf Python 3.7, formatiert aber in unserem Fall Python 2.7 Sourcecode.

Da black Strings etwas anders formatiert, als weiter unten eingesetzte Tools, lassen wir das hier mti dem sTring formatieren. Bonus: Black hat [schöne Tips zum umgang mit git blame](https://github.com/psf/black#migrating-your-code-style-without-ruining-git-blame).

## pre-commit-hooks

pre-commit selbst kommt mit einer Menge [praktischer Hooks](https://github.com/pre-commit/pre-commit-hooks/blob/master/README.md) daher:


    - repo: https://github.com/pre-commit/pre-commit-hooks
      rev: v2.0.0
      hooks:
        - id: check-ast
        - id: fix-encoding-pragma
        - id: trailing-whitespace
        - id: double-quote-string-fixer
        - id: check-docstring-first
        - id: check-builtin-literals
          args: ['--no-allow-dict-kwargs']
        - id: requirements-txt-fixer
        - id: check-yaml
        - id: check-json
        - id: pretty-format-json
          args: ['--autofix']
        - id: check-xml
        - id: check-merge-conflict
        - id: check-case-conflict
        - id: check-symlinks
        - id: name-tests-test
        - id: detect-private-key
        - id: end-of-file-fixer
        - id: mixed-line-ending
        - id: check-executables-have-shebangs
        - id: flake8
          additional_dependencies:
            - flake8-docstrings
            - flake8-comprehensions
            - flake8-tuple
            # flake8 uses tox.ini

`check-ast` führt einen schnellen Check auf Syntax-Errors durch. `fix-encoding-pragma` sorgt dafür, dass alle Dateien mit `# -*- coding: utf-8 -*-` beginnen. Kann man beim Umstieg auf Python 3 wegmachen. `trailing-whitespace` beseitigt Leerzeichen am Zeilenende - das sorgt oft für Verwirrung bei Git-Merges. `double-quote-string-fixer` erzwingt einfache Anführungszeichen (black hätte ja lieber doppelte).

`check-docstring-first` stellt sicher, dass kein code vor dem Docstring kommt, `check-builtin-literals` erzwingt einheitliche Initialisierung. In insbesondere muss ja `dict(a='b')` und `{'a': 'b'}` nicht wild gemischt werden. Das ganze macht aber bei Unicode und Python 2.7 manchmal recht unerwartete Sachen, weil die Objekt-Literal ja Unicode Keys erzeugt.

`requirements-txt-fixer` sortiert `requirements.txt` und vermeidet damit Merge-Konflikte. `check-yaml`, `check-json`, `check-xml` prüfen die entsprechenden Dateien auf Syntax-Fehler. `pretty-format-json` bringt JSON-Dateien in ein einheitliches Format - wobei das Sortieren der Keys da nicht immer die beste Lösung ist, aber einen Tod muss man sterben.

`check-merge-conflict` sorgt dafür, dass man nicht vergessene Merge-Conflict-Marker comittet, `check-case-conflict` erwischt Dateien, die auf MacOS Dateisystemen Ärger machen. `check-symlinks` vermeidet tote Symlinks. `name-tests-test` versucht sicherzustellen, dass Unittests einheitlich benannt sind - mag je nach Projektstruktur keine gute Idee sein. 

`detect-private-key` versucht sicherzustellen, dass nicht Kram in git landet, der da nicht hin gehört. 

`end-of-file-fixer` und `mixed-line-ending` versucht gemischtes Unix/Windows Dateiformat im Griff zu behalten.

`check-executables-have-shebangs` stellt sicher, dass man nicht aus Versehen irgendwas mit `+x` im Dateisystem markiert, was kein Executable ist.

## flake8

`flake8` ist ein Riesending, das auch von den `pre-commit-hooks` ausgeführt wird. Wir wollen hier nicht jeden möglichen Programmierfehler angezeigt bekommen, sondern nur sicherstellen, dass veralteter oder offensichtlich falscher Code nicht comittet wird. Das richtige Gleichgewicht in der Konfiguration ist da von Projekt zu Projekt verschieden. 

Die eigentliche Konfiguration findet sich in `tox.ini`

[flake8-comprehensions](https://github.com/adamchainz/flake8-comprehensions) findet viele veraltete Konstrukte rund um List-, Dict- und Set-Comprehensions. [flake8-tuple](https://github.com/ar4s/flake8_tuple) erwischt den viel zu häufig auftretenden Fehler in `foo = 123,`, der oft albtraumhaft zu debuggen ist.

[flake8-docstrings](https://gitlab.com/pycqa/flake8-docstrings) koppelt [pydocstyle](https://github.com/pycqa/pydocstyle) ein. Damit wird die Existenz von Docstrings /erzwungen/. Das ist nicht schön, wenn man kleine Änderungen in großen legacy-Files macht. Aber dafür gibt es ja `gin commit -n`.

[flake8](https://flake8.pycqa.org) und die Plugins werden in `tox.ini` konfiguriert. Etwa so:

    [flake8]
    # D105 Missing docstring in magic method
    # D107 Missing docstring in __init__
    # D203 1 blank line required before class docstring (found 0) - ⚡️ black
    # D207 Docstring is under-indented - ⚡️ black
    # D213 "Multi-line docstring summary should start at the second line" - ⚡️ black
    # D401 First line should be in imperative mood; try rephrasing - ⚡️ deutsch
    # D402 - false positives
    # D403 First word of the first line should be properly capitalized - ⚡️ deutsch
    # D406 Section name should end with a newline ('Returns', not 'Returns:') -     false positives
    # D407 Missing dashed underline after section ('Returns') - false positives
    # E203 whitespace before - ⚡️ black
    # E123 closing bracket does not match indentation of opening bracket's line - ⚡️     black
    # N801 CapsWord
    # W503 line break before binary operator - ⚡️ black
    # W504 line break after binary operator - ⚡️ black
    # W606 'async' and 'await' are reserved keywords starting with Python 3.7
    ignore = D105, D107, D203, D207, D213, D401, D402, D403, D406, D407, E123, E203, N801, W503, W504
    # extend-ignore =
    exclude = tests/conftest.py, lib/appengine-toolkit2/gaetk2/tools/unicode.py, lib/    appengine-toolkit2/gaetk2/vendor
    builtins = _
    
    [pydocstyle]
    # see .vscode/settings.json
    ignore = D104, D105, D203, D207, D213, D401, D402, D403, D406, D407
    max-line-length = 110
    match=(?!test_).*(?!_test)\.py

Ein paar Sachen kommen nicht gut mit der deutschen Sprache zurecht oder beissen sich mit der Art, wie `black` den Code formatiert.


## python-modernize

Als nächstes stellt [python-modernize](https://python-modernize.readthedocs.io/en/latest/fixers.html) sicher, das diverse Konstrukte Python 3 freundlicher werden. 

    - repo: https://github.com/python-modernize/python-modernize
      rev: a234ce4e185cf77a55632888f1811d83b4ad9ef2
      hooks:
        - id: python-modernize
          args:
            - --fix=ws_comma
            - --fix=set_literal
            - --fix=print
            - --fix=idioms
            - --fix=default
            # - --future-unicode
            - --no-six
            - --nobackups
            - -w

`file()` wird zu `open()`, `i.next()` zu `next(i)`, Importe werden absolut und der moderne `raise()` Syntax wird verwendet ([Dokumentation](https://python-modernize.readthedocs.io/en/latest/fixershtml#fixers-with-no-dependencies)). Unnötiger Whitespace hinter Kommata [kommt weg](https://docs.python.org/3/library/2to3.html#2to3fixer-ws_comma), Set-Literals [werden verwendet](https://docs.python.org/3/library/2to3.html#2to3fixer-set_literal), und noch so einges mehr. Siehe [hier](https://python-modernize.readthedocs.io/en/latest/fixers.html#to3-fixers) und [hier](https://docs.python.org/3/library/2to3.html#2to3fixer-idioms). Das sollte alles am ende immer noch Python 2.7 kompatibel sein, aber schon deutlich mehr Richtung modernes Python gehen.

## python-import-sorter

[python-import-sorter](https://github.com/FalconSocial/pre-commit-python-sorter) verheiratet pre-commit mit [isort](https://timothycrosley.github.io/isort/).

    - repo: git://github.com/FalconSocial/pre-commit-python-sorter
      rev: b57843b0b874df1d16eb0bef00b868792cb245c2
      # uses tox.ini
      hooks:
        - id: python-import-sorter

`isort` sorgt dafür, dass die Imports in Python-Dateien immer einheitlich sortiert und formatiert sind. 


Auch hier findet sich die eigentliche Konfiguration in `tox.ini`:

    [isort]
    force_alphabetical_sort_within_sections = true
    force_single_line = true
    lines_between_types = 1
    lines_after_imports = 2
    add_imports = from __future__ import unicode_literals
    force_to_top = commandlinetools, config
    known_standard_library = commandlinetools, config, typing
    known_third_party = cs, gaetk, gaetk2, google, huTools, wtforms_appengine
    known_first_party = modules common
    virtual_env = .
    skip=appengine_config.py, tests/conftest.py

Einiges ist Geschmacksache. Wichtig ist, dass es oft bootstrap.module und der gleichen gibt, die man /nicht/ sortierend darf, die landen in `skip`. Konfigurationsmodule müssen oft als erstes importiert werden, die landen in `force_to_top`. Mit den `known_` Optionen, kann man isort helfen, die Module sinnvoll in Blöcke zu unterteilen. `virtual_env` hilft mit unserem spezifischen App Engine Layout.

## yamllint

Google Appengine aber auch CircleCI und so manches andere verwendet YAML Konfigurationsdateien. Ab einer gewissen Komplexität kann da einiges schief laufen. [yamllint](https://yamllint.readthedocs.io/en/stable/) erwischt die gröbsten Schnitzer.

    - repo: https://github.com/adrienverge/yamllint.git
      rev: v1.15.0
      hooks:
        - id: yamllint
          args:
            - '-d {extends: default, rules: {
                document-start: {present: false},
                octal-values: {forbid-implicit-octal: true},
                trailing-spaces: {},
                truthy: {},
                line-length: {max: 110, level: warning, allow-non-breakable-words:   true, allow-non-breakable-inline-mappings: true}}}'

## JavaScript Standard Style

[JavaScript Standard Style](https://standardjs.com) ist die dreiste Behauptung, dass es einen Konsens über JS Codeformat gibt. Gibt es nicht, aber was StandardJS hat gutes Tooling und entspricht einigermaßen den Konventionen von Python Programmierern. Also scheuchen wir unseren gesamten Javascript-Code durch diesen Formatter:

  - repo: https://github.com/mdornseif/standardjs-mirror.git
    rev: 7fda18f6d6608c1424d8253eb9a0f1b668c4a3ef
    hooks:
      - id: standard
        args:
          - --parser babel-eslint
          - --fix
        exclude: >
              (?x)(
                  /js_src/vendor/|
                  /lib/(py2js|gae_mini_profiler|appengine-toolkit)/|
                  /static/js/kalkulation.js$
              )
        always_run: false
        # log_file: standard.log

Damit das ganze rund läuft, muss man noch einmal mittels `yarn add babel-eslint` eine `package.json` Datei erzeugen.

# Fazit

Mit diesem pre-commit setup, dauert jeder Commit etwas länger (weil ja viele Überprüfungen laufen), aber viele Fehlerquellen werden schon an der Wurzel ausgeschlossen. Damit entlastet man den CI-Error-Debug-Zyklus erheblich.

