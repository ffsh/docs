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

Das :code:`build.py` Skript ist ein in Python geschriebenes Programm zum Bauen der Gluon Firmware. Es ist nicht Perfekt und könnte ein Reafctoring gebrauchen aber es erledigt seinen Job.
Im folgenden werden zunächst zwei Tabellen gezeigt und dann anhand eines Beispiels das Bauen der Firmware präsentiert.

Tabellen
~~~~~~~~

:code:`build.py` unterstützt verschiedene Befehle (commands):

+---------+----------+-------------------+-------------------------------------------------+
| command | value    | make "equivalent" | Kommentar                                       |
+---------+----------+-------------------+-------------------------------------------------+
| -c      | update   | make update       | lädt opwenwrt und wendet gluon patches an       |
+---------+----------+-------------------+-------------------------------------------------+
| -c      | build    | make build        | baut die firmware                               |
+---------+----------+-------------------+-------------------------------------------------+
| -c      | clean    | make clean        | löscht alle packages des targets                |
+---------+----------+-------------------+-------------------------------------------------+
| -c      | dirclean | make dirclean     | löscht alle targets und die toolchain           |
+---------+----------+-------------------+-------------------------------------------------+
| -c      | sign     | n/a               | signiert die firmware                           |
+---------+----------+-------------------+-------------------------------------------------+

Außerdem gibt es eine Reihe von Argumenten.

+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| command  | value                          | default     | name         | pflicht | Kommentar                                                                              |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -b       | dev or testing or rc or stable | dev         | Branch       | ja      | der Firmware branch                                                                    |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -w       | site                           | n/a         | Workspace    | ja      | Pfad zum site Repository                                                               |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -n       | 42                             | n/a         | Build Number | ja      | build Nummer wird von jenkins automatisch hochgezählt wird im firmware Namen verwendet |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -t       | ar71xx-generic or ...          | all targets | Target       | nein    | ohne Angabe werden alle Targets gebaut, mit Angabe nur der angegebene Target           |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -s       | <pfad zu secret>               | n/a         | Secret       | nein    | wird nur beim signieren benötigt                                                       |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| -d       | <pfad zu public directory>     | n/a         | Directory    | nein    | wird nur bei -publish benötigt                                                         |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| --commit | der verwendete commit          | n/a         | Commit       | ja      | commit sha, dient als Referenz im build.json                                           |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| --cores  | 1 bis N                        | 1           | Cores        | nein    | Anzahl der zu verwenden Threads, Empfehlung: CPU-Kerne+1                               |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+
| --log    | V=w or V=s                         |   /       | Log          | ja    | Log level w: nur Warnungen/Fehler, s: alles                                            |
+----------+--------------------------------+-------------+--------------+---------+----------------------------------------------------------------------------------------+

Beispiel
~~~~~~~~

.. code-block:: bash
    :linenos:

    git clone --recurse https://github.com/ffsh/site.git
    cd site
    ./build.py -c update -b grotax -n 1 -w $(pwd) --commit $(git rev-parse HEAD) --log "V=w"
    ./build.py -c build -t "ar71xx-tiny" -b grotax -n 1 -w $(pwd) --commit $(git rev-parse HEAD) --log "V=w"

In Zeile 1 wird das Repository inklusive der "submodules" geklont. Danach in Zeile 2 wechseln wir in das "site" Verzeichnis.
Dort führen wir zum ersten mal das :code:`build.py` Skript aus (Zeile 3).

- :code:`-c update` (Wir wollen die Abhängigkeiten von Gluon aktualisieren)
- :code:`-b grotax` (Hier kann ein beliebiger Name eingesetzt werden)
- :code:`-n 1` (Es ist unser erster build)
- :code:`-w $(pwd)` (Der Workspace in diesem Fall das Aktuelle Verzeichnis (pwd))
- :code:`--commit $(git rev-parse HEAD)` (Der Commit-Hash wird hier ermittelt)

In Zeile 4 besteht der unterschied dann nur in dem :code:`-c build` (wir wollen nun bauen) und dem :code:`-t "ar71xx-tiny"` (hier wird nur für ein target gebaut).

Mirror
------
Die Firmware wird zentral auf einer Storage Box von Hetzner gehostet. Diese Storage Box wird als Netzwerklaufwerk eingebunden.

Als root-User oder mit sudo

::

    nano /etc/fstab

Die Datei um folgenden Eintrag ergänzen:

::

    //u205465.your-storagebox.de/backup /mnt/firmware       cifs    iocharset=utf8,rw,credentials=/etc/firmware-credentials.txt,uid=0,gid=0,file_mode=0644,dir_mode=0744 0       0

Dann erstellen wir die Credentials-Datei.
::

    nano /etc/firmware-credentials.txt

Inhalt so wie hier Daten gibts beim NOC

:: 

    username=
    password=

Rechte anpassen, cifs installieren und einbinden
::

    chmod 600 /etc/firmware-credentials.txt
    apt install cifs-utils
    systemctl daemon-reload
    systemctl restart remote-fs.target

Jetzt hast du ein Netzwerklaufwerk :)

::

    ls /mnt/firmware

Für nginx gibt es eine vorbereitete Konfiguration.

.. literalinclude:: configs/nginx-firmware.conf

jenkins
-------

Jenkins Projekt Bash Befehle

::

    export PYTHONUNBUFFERED=1
    ./build.py -c update -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT} --log "V=w"

    # Befehle die nur manchmal notwenig sind
    #./build.py -c dirclean -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT}
    #./build.py -c clean -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT}

    ./build.py -c build -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT} --cores "9" --log "V=w"
    ./build.py -c sign -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT} -s ${SECRET}
    ./build.py -c publish -b ${GIT_BRANCH} -n ${BUILD_NUMBER} -w ${WORKSPACE} --commit ${GIT_COMMIT} -d "/var/www/firmware.grotax.de"


Changelog
---------

Dieses Changelog stellt nur die durch uns durchgeführten Änderungen da. Für Änderungen an Gluon (bsp unterstützte Geräte) verweisen wir auf die Dokumentation von Gluon.
Die Versionierung entspricht der Versionierung von Gluon. Dieses Changelog bezieht sich auf den :code:`stable` Zweig die anderen Zweige können davon abweichen.

2018.2.1 build xxx
~~~~~~~~~~~~~~~~~~


2018.2 build 163
~~~~~~~~~~~~~~~~
- fix: fehlendes Paket für die vpn-Konfiguration

2018.1.1 build 141
~~~~~~~~~~~~~~~~~~
- release ohne viele Änderungen (bereits in >=134 enthalten)
- fehlendes Paket für die vpn Konfigruation

2018.1 build 134
~~~~~~~~~~~~~~~~
- erste gemeinsame version für lauenburg und stormarn
- einührung der drei domains ffsh, ffod, ffrz

    - Freifunk Südholstein: ffsh
    - Freifunk Stormarn: ffod
    - Freifunk Lauenburg: ffrz
- bereits mit patches für autoupdater
- mit Einführung der neuen Karte wurde alfred entfernt (auch in 2017.1.x)

2018.1 build < 131
~~~~~~~~~~~~~~~~~~
- Fehler im Autoupdater