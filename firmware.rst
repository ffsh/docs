Firmware
========

site-repo
---------

Struktur
~~~~~~~~

Es gibt vier Hauptzweige

1. stable
2. rc
3. testing
4. dev

Dabei wird stets angestrebt von oben nach unten eine immer stabilere Firmware zu erstellen. Um dies sicherzustellen kommen bei :code:`stable` überwiegend Gluon point-releases zum Einsatz also z.B. :code:`v2018.1`. Der Zweig :code:`rc` ist wiederum dazu gedacht, Gluon point-releases vor der Veröffentlichung in :code:`stable` zu testen. Der Zweig :code:`testing` basiert stets auf einem Gluon main-release also :code:`v2018.1.x`. Während die point-releases als tag (git tag) auf einen bestimmten Commit zeigen und sich nicht mehr ändern, handelt es sich bei den main-releases um Zweige die regelmäßig Updates erhalten. Daher erhält der :code:`testing` Zweig auch häufiger Updates. Der letzte Zweig ist :code:`dev` dieser wird verwendet, um den :code:`master` oder :code:`next` Zweig von Gluon in unserer Umgebung frühzeitig zu testen. Er erhält nach Bedarf Updates.

Für das Git-Repository bedeutet dies, dass Änderungen in der Regel zu erst in :code:`dev` landen. Die anderen Zweige basieren auf :code:`dev` und haben nur einen weiteren Commit der die nötigen Anpassungen (andere Gluon-Version) enthält.

Versionierung
-------------
Während wir die Gluon-Versionierung übernehmen fügen wir eine "build number" als auch den Zweig zur Version hinzu.
::

    gluon-ffsh-$gluonVersion-$buildNumber-$Zweig-$Gerät
    gluon-ffsh-2018.1.1-147-dev-tp-link-archer-c7-v4-sysupgrade.bin

- $gluonVersion: entspricht der Gluon-Version
- $buildNumber: wird mit jedem versuch eine Firmware zu bauen hochgezählt
- $Zweig: stable, rc, testing, dev
- $Gerät: das entsprechende Gerät

Die Gluon-Version bietet den eindeutigen Vorteil das auf einen Blick erkennbar ist welche Version von Gluon auf dem Gerät läuft.
Die "build number" sorgt dafür, dass innerhalb einer Gluon-Version mehrere Versionen verteilt werden können. Dies kann Nötig werden wenn ein Paket vergessen wurde.
Den Zweig in der Version zu haben, lässt den Betrachter sofort erkennen, mit welcher Version er es zu tun hat.

Die Reihenfolge wurde gewählt um Updates von einen Zweig auf den anderen zu ermöglichen ob dies empfehlenswert ist, steht auf einem anderen Blatt. Da es jedoch in der Vergangenheit zu wechseln des Zweiges kam wurde dies als wichtige Funktion betrachtet.

Wenn in Zukunft ein neuer Branch eingeführt würde könnten mit dem :code:`au-changer` Paket alle Knoten auf dem alten Zweig auf diesen eingestellt werden. Und es wäre nicht abhängig vom Namen oder der Gluon-Version ob ein update erfolgt oder nicht.
Dieser extra Aufwand ist notwendig weil der autoupdater in Gluon sehr einfach aufgebaut ist und die Zweige keine Gewichtung oder ähnliches haben. Ob ein Update erfolgt oder nicht, wird durch Vergleichen der eigen Version, mit der Version im Dateinamen entschieden. Dabei kommt im Grunde nur ein :code:`<` Vergleich zum Einsatz.

build.py
--------

jenkins
-------

Changelog
---------

Dieses Changelog stellt einen groben überblick über die Änderungen in der Firmware dar.
Die Versionierung entspricht der versionierung von Gluon. Dieses Changelog bezieht sich auf den :code:`stable` Zweig die anderen Zweige können davon abweichen.

2018.1.1 ab build xxx
~~~~~~~~~~~~~~~~~~~~~
- fix: fehlendes Paket für die vpn Konfigruation

2018.1.1 ab build 141
~~~~~~~~~~~~~~~~~~~~~
- release ohne viele Änderungen (bereits in >=134 enthalten)
- fehlendes Paket für die vpn Konfigruation


2018.1 ab build 134
~~~~~~~~~~~~~~~~~~~
- erste gemeinsame version für lauenburg und stormarn
- einührung der drei domains ffsh, ffod, ffrz

    - Freifunk Südholstein: ffsh
    - Freifunk Stormarn: ffod
    - Freifunk Lauenburg: ffrz
- bereits mit patches für autoupdater
- mit Einführung der neuen karte wurde alfred entfernt (auch in 2017.1.x)

2018.1 build < 131
^^^^^^^^^^^^^^^^^^
- Fehler im Autoupdater