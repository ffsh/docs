.. toctree::
   :maxdepth: 2
   :caption: Inhalte

meshviewer
==========

Die Folgenden Schritte dienen dazu, eine Karte anzuzeigen. Dies lässt
sich auch auf einem vom Gateway getrennten System durchführen. ### Karte

[[Datei:map-uebersicht.png]]

yanic
-----

`yanic <https://github.com/FreifunkBremen/yanic>`__ sammelt von den
Knoten Daten, welche dann auf einer Karte angezeigt werden können,
früher wurde hierfür Alfred benutzt. yanic ist in go geschrieben also
installieren wir eine neue Version von go.
`golang <https://golang.org/dl/>`__

::


   wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz
   # Bitte sha256 vergleichen
   tar -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz
   rm go1.10.1.linux-amd64.tar.gz


::


   sudo su


Als root in ~/.bashrc

::


   GOPATH=/opt/go
   PATH=$PATH:/usr/local/go/bin:$GOPATH/bin


Mit whereis go prüfen ob go gefunden wird:

::


   go: /usr/local/go /usr/local/go/bin/go


Dann wird yanic installiert.

::


   go get -v -u github.com/FreifunkBremen/yanic


Die Konfiguration von Yanic wird in /etc/yanic.conf angelegt. Eine
Beispiel gibt es
`hier <https://raw.githubusercontent.com/ffsh/ffshConfigs/master/yanic.conf>`__:

Wir können testen ob yanic funktioniert in dem wir eine manuelle Anfrage
stellen hier an das Gateway Hopfenbach:

::


   yanic query --wait 5 bat0 "fddf:0bf7:80::128:1"


Damit yanic auch als Deamon läuft legen wir noch einen service an.

::

    sudo cp /opt/go/src/github.com/FreifunkBremen/yanic/contrib/init/linux-systemd/yanic.service /lib/systemd/system/yanic.service
    sudo systemctl daemon-reload

influxdb
--------

Influxdb dient als Datenbank für yanic

::


   sudo apt install influxdb influxdb-client


Nun sichern wir die influxdb ab /etc/influxdb/influxdb.conf

Hier werden nur die empfohlenen Anpassungen beschrieben: Noch vor der
[meta] Sektion setzen wir, sonst wäre der port 8088 überall offen.

::


   bind-address = "localhost:8088"


Weiter unten bei [admin] das gleiche:

::


   bind-address = "localhost:8083"


kurz danach in [http]

::


   bind-address = "localhost:8086"


systemctl restart influxdb

Nun sollte influxdb nur noch auf localhost erreichbar sein, prüfen kann
man dies mit netstat -tlpn

Grafana
-------

Grafana kann Graphen erstellen welche im meshviewer eingebunden werden
können. Hier wird
`Grafana <http://docs.grafana.org/installation/debian/>`__ über eine
Repository installiert.

::


   deb https://packagecloud.io/grafana/stable/debian/ stretch main
   curl https://packagecloud.io/gpg.key | sudo apt-key add -
   sudo apt-get update
   sudo apt-get install grafana


TODO sichere Konfiguration

meshviewer
----------

Für den Meshviewer installieren wir als erstes nodejs und yarn

nodejs
~~~~~~

Wir brauchen ein aktuelles nodejs das finden wir auf
`nodejs.org <https://nodejs.org/en/download/package-manager/>`__ Wir
benutzen die LTS Variante 8.x

::


   curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
   sudo apt-get install -y nodejs


yarn
~~~~

Dann installieren wir
`yarn <https://yarnpkg.com/en/docs/install#linux-tab>`__

::


   curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
   echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list


meshviewer-rgb
~~~~~~~~~~~~~~

Nun installieren wir den `meshviewer <https://doc.meshviewer.org/>`__
selbst. Im web Verzeichnis /var/www/

::


   git clone https://github.com/ffsh/meshviewer.git
   cd meshviewer
   yarn


Nun muss die Konfiguration in meshviewer/config.js eventuell noch
angepasst werden.

Danach yarn run gulp Nun muss nur noch ein Webserver meshviewer/build
ausliefern.
