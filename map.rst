.. toctree::
   :maxdepth: 2
   :caption: Inhalte

map
===

Diese Dokumentation behandelt das Aufsetzen von https://map.freifunk-suedholstein.de

.. image:: images/meshviewer-concept.png

Vorraussetzungen
----------------

Wie für all unsere Service Server verwenden wir Ubuntu als Basis.
Als Hardware verwenden für den gesammten Stack eine VM von Hetzner (CPX21) 2vCores 4GB RAM 40GB disk.
Für unsere Community reicht dieses Setup bisher aus.

yanic
-----

`yanic <https://github.com/FreifunkBremen/yanic>`__ sammelt von den
Knoten Daten, welche dann auf einer Karte angezeigt werden können,
früher wurde hierfür Alfred benutzt. yanic ist in go geschrieben also
installieren wir eine neue Version von go.
`golang <https://golang.org/dl/>`__


::


   wget https://dl.google.com/go/go1.17.5.linux-amd64.tar.gz
   # Bitte sha256 vergleichen https://golang.org/dl/
   tar -C /usr/local -xzf go1.17.5.linux-amd64.tar.gz
   rm go1.17.5.linux-amd64.tar.gz


In :code:`~/.bashrc`

::


   GOPATH=/opt/go
   PATH=$PATH:/usr/local/go/bin:$GOPATH/bin

Hier musst du dich einmal abmelden und neu anmelden damit die Variablen auch gesetzt werden.

Nach dem Anmelden kann man prüfen ob die Variablen korrekt gesetzt wurden.

::


  echo $GOPATH
  /opt/go


Mit :code:`whereis` go prüfen ob go gefunden wird:

::


   go: /usr/local/go /usr/local/go/bin/go


Dann wird yanic installiert.

::


   go get -v -u github.com/FreifunkBremen/yanic


Die Konfiguration von Yanic wird in :code:`/etc/yanic.conf` angelegt:


.. literalinclude:: configs/yanic.conf

Wir können testen ob yanic funktioniert in dem wir eine manuelle Anfrage
stellen hier an das Gateway Hopfenbach:

::


   yanic query --wait 5 br-ffsh "fddf:bf7:80::48:1"


Damit yanic auch als Deamon läuft legen wir noch einen Service an.

::

    sudo cp /opt/go/src/github.com/FreifunkBremen/yanic/contrib/init/linux-systemd/yanic.service /lib/systemd/system/yanic.service
    sudo systemctl daemon-reload

influxdb
--------

Influxdb dient als Datenbank für :code:`yanic`

Achtung hier wird Influxdb 1.x (aktuell 1.8.10) installiert, die aktuelle version ist 2.0 (diese wird aktuell nicht von yanic unterstützt).
::


   wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
   source /etc/lsb-release
   echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

::


   sudo apt install influxdb influxdb-client


Nun sichern wir die influxdb ab :code:`/etc/influxdb/influxdb.conf`

Hier werden nur die empfohlenen Anpassungen beschrieben: Noch vor der
:code:`[meta]` Sektion setzen wir, sonst wäre der port 8088 überall offen.

::


   bind-address = "localhost:8088"


Weiter unten bei :code:`[admin]` das gleiche:

::


   bind-address = "localhost:8083"


kurz danach in :code:`[http]`

::


   bind-address = "localhost:8086"


::

   systemctl restart influxdb

Nun sollte influxdb nur noch auf localhost erreichbar sein, prüfen kann
man dies mit :code:`netstat -tlpn`

Grafana
-------

Grafana kann Graphen erstellen welche im meshviewer eingebunden werden
können. 

Folge für die Installation der offiziellen `Grafana Dokumentation <http://docs.grafana.org/installation/debian/>`__.

Da Grafana bei uns hinter einem Proxy laufen soll, setzen wir auch hier alle IPs auf :code:`localhost`.
Am besten einmal am Ende prüfen ob alles richtig konfiguriert ist mit :code:`netstat -tlpn`.

Ein wichtiger Punkt ist der öffentliche Zugang, damit die Statistiken auch von Besuchern abgerufen werden können.

::


   #################################### Anonymous Auth ##########################
   [auth.anonymous]
   # enable anonymous access
   enabled = true

   # specify organization name that should be used for unauthenticated users
   org_name = Freifunk Südholstein

   # specify role for unauthenticated users
   org_role = Viewer

Die Organisation kann man als Admin in Grafana anlegen.

meshviewer
----------
Für die Karte muss meshviewer installiert werden `meshviewer <https://github.com/freifunk/meshviewer>`__

In :code:`/var/www/` ein neues Verzeichnis :code:`map` anlegen.
In :code:`/var/www/map` ein neues Verzeichnis :code:`meshviewer` an.

Unter `Releases <https://github.com/freifunk/meshviewer/releases>`__ meshviewer-build.zip in :code:`meshviewer` herunterladen.

Zusätzlich muss eine config.json in dem gleichen Verzeichnis abgelegt werden.

.. literalinclude:: configs/meshviewer-config.json

Dann muss nginx konfiguriert werden.

.. literalinclude:: configs/nginx-map.conf

Damit die Karte ein Bild der Router anzeigen kann wird innerhalb von :code:`/var/www/map` das Repository device-pictures geklont.

::

   git clone https://github.com/freifunk/device-pictures.git

Tile-cache mit nginx
---------------------

Für den Meshviewer benötigt man einen Tile-Server, der die Karte als einzelne Kacheln ausliefert.
Wir verwenden dabei das kostenlose und freie Angebot von OpenStreetMap. Damit die Server von OpenStreetMap weniger stark belastet werden verwenden wir einen Tile-Cache. Bei einer Anfrage für eine Karten-Kachel fragt der Browser den Cache, hat dieser die Datei bereits, so liefert er sie direkt aus. Hat er sie nicht, so fragt er bei den OpenStreetMap Servern und speichert die Datei in seinem Cache.
Für die einfache Umsetzung haben ein paar Freifunker an einer Konfiguration für nginx gearbeitet, welche genau das umsetzt.

Voraussetzungen:

- nginx erreichbar unter der entsprechenden Domain
- TLS mit gültigem Zertifikat (Let's Encrypt)
- ein wenig Speicherplatz


.. literalinclude:: configs/nginx-tilecache.conf

Grafna cache mit nginx
----------------------

Da Grafana ab Version 7.0 das rendern der images, welche wir auf der Karte einbetten, anders rendert als früher mussten wir einen Cache einrichten.
Siehe https://github.com/ffrgb/meshviewer/issues/304

Da Grafana keinen image renderer mehr enthält muss ein plugin installiert werden: grafana-image-renderer

Basierend auf den Kommentaren haben wir auch eine Konfiguration zusammengestellt.


.. literalinclude:: configs/nginx-grafana.conf
