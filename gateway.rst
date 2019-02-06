.. toctree::
   :maxdepth: 2
   :caption: Inhalte

Gateway Konfiguration
=====================

Wenn du ein neues Gateway einrichten willst, solltest du dir als erstes ein freies aus der Tabelle :ref:`infrastruktur-gateways` aussuchen.
Du könntest auch ein neues Gateway definieren, dieses muss natürlich zur restlichen Infrastruktur passen und in der Firmware eingetragen werden.
Für die Einrichtung brauchen wir dann folgende Daten:

============ ======= ==========
Datum        Kürzel  Kommentar
------------ ------- ----------
fastd-MAC    $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC   $MACb   Die MAC-Adresse für das Batman Inteface
IPv4         $IPv4   Feste IPv4 für Freifunk
IPv6         $IPv6   Feste IPV6 für Freifunk
Gateway-Name $gwName Name deines Gateways
Public-IPv4  $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6  $IPv6P  Öffentliche feste IPv6 deines Gateways
============ ======= ==========

Zu der :code:`$IPv4` und :code:`$IPv6` gehören natürlich noch jeweils das passende Netzwerk, welches an die Clients verteilt wird. Wähle also passende Subnetze aus und dazu gehörende Adressen für dein Gateway. Für unsere vordefinierten Gateways sind diese Daten bereits festgelegt.

Als nächstes solltest du dir Hardware oder eine VM suchen, auf der dein Gateway laufen soll.
Dabei gibt es eine zwingende Voraussetzung, denn du musst in der Lage sein einen eigenen Kernel zu verwenden, informiere dich also vorher ob du bei der eingesetzten Virtualisierungslösung einen eigenen Kernel verwenden kannst. Bei `KVM <https://de.wikipedia.org/wiki/Kernel-based_Virtual_Machine>`_ geht das ohne Probleme.

Die sonstigen Spezifikation hängen stark von der Anzahl der Knoten und Clients in deinem Netzwerk ab. Bei uns reichen aktuell die günstigen VMs aus der Hetzner Cloud.

Als Betriebssystem hat sich Debian etabliert, die Anleitung setzt auf Debian 9.

Wenn du den Traffic der Clients nicht direkt an deinem Gateway ins Internet lassen möchtest, dann solltest du dir auch einen VPN-Anbieter suchen. Wir haben gute Erfahrungen mit Mullvad und Private Internet Access gemacht aber es sollten auch andere VPN-Anbieter funktionieren.

Nach der Standard Debian Installation (minimal reicht). Verbindest du dich mit deinem Gateway. Dann wechselst du falls Notwendig in den :code:`root` Nutzer.

Allgemeines
-----------

Zum Einstieg installieren wir einige Pakete die wir später brauchen werden.

::

   apt install build-essential git apt-transport-https bridge-utils ntp net-tools

Wir legen auch einen neuen Nutzer an.

::

   adduser ffsh
   usermod -aG sudo ffsh

VPN-Exit || RAW-Exit
--------------------

Spätestens jetzt solltest du dich entschieden haben ob du einen VPN-Anbieter verwenden möchtest (VPN-Exit) oder nicht (RAW-Exit).
Die Meisten Anbieter werden eine Anleitung für Debian haben, wenn nicht solltest du vielleicht einen anderen wählen.

Wenn du einen VPN-Exit erstellen möchtest, dann solltest du auch noch herausfinden wie du bei "on up" und "on down" jeweils ein Skript ausführen kannst.

Wir erweitern die Tabelle um zwei Einträge:

1. Das Internet-Interface deines Servers.
2. Das Exit-Interface, wenn du ein VPN-Exit erstellst, dann ist dies dein VPN-Interface. Wenn du allerdings ein RAW-Exit machst, dann trägst du hier auch das Internet-Interface ein.

================== ======= ==========
Datum              Kürzel  Kommentar
------------------ ------- ----------
fastd-MAC          $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC         $MACb   Die MAC-Adresse für das Batman Inteface
IPv4               $IPv4   Feste IPv4 für Freifunk
IPv6               $IPv6   Feste IPV6 für Freifunk
Gateway-Name       $gwName Name deines Gateways
Public-IPv4        $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6        $IPv6P  Öffentliche feste IPv6 deines Gateways
Internet-Interface $InetI  Das Internet-Interface deines Servers
Exit-Interface     $ExitI  Das Exit-Interface
================== ======= ==========

Batman
------

Freifunk Südholstein Benutzt für das Mesh-Netzwerk batman-adv und den Routing-Algorithmus Batman IV. Für die VPN-Tunnel zu den Knoten benutzen wir Fastd.
Da das Batman-Kernel-Modul nicht in einer aktuellen Version im OS-Repository ist, müssen wir es selber bauen.

batman-adv und batctl
~~~~~~~~~~~~~~~~~~~~~

Für Batman gibt es ein Kernel-Modul und ein Kontroll-Programm. Als erstes installieren wir aber ein paar Pakete die wir benötigen.

::

    apt update && apt upgrade

    apt install linux-headers-amd64 sudo

    apt install libnl-3-dev libnl-genl-3-dev libcap-dev pkg-config dkms

Damit bei einem Kernel-Update automatisch ein neues Kernel-Modul gebaut wird, benutzen wir :code:`dkms`.
Wir wechseln das Verzeichnis, laden batman-adv herunter und legen die Konfiguration an.
::

    cd /usr/src
    wget https://downloads.open-mesh.org/batman/releases/batman-adv-2018.4/batman-adv-2018.4.tar.gz
    tar xfv batman-adv-2018.4.tar.gz
    cd batman-adv-2018.4/
    nano dkms.conf

Die :code:`dkms.conf` befüllen wir mit dem folgenden Inhalt:

::

    PACKAGE_NAME=batman-adv
    PACKAGE_VERSION=2018.4

    DEST_MODULE_LOCATION=/extra
    BUILT_MODULE_NAME=batman-adv
    BUILT_MODULE_LOCATION=net/batman-adv

    MAKE="'make' CONFIG_BATMAN_ADV_BATMAN_V=n"
    CLEAN="'make' clean"

    AUTOINSTALL="yes"

Danach prüfen wir ob alles funktioniert.
Mit dem ersten Befehl registrieren wir das Modul bei dkms. Mit dem zweiten Befehl wird das Modul kompiliert.
Der letzte Befehl installiert das Modul für den aktiven Kernel.

::

    dkms add -m batman-adv -v 2018.4
    dkms build -m batman-adv -v 2018.4
    dkms install -m batman-adv -v 2018.4

Wenn es bei einem dieser Schritte zu Problemen kommt, dann prüfe noch mal deine Konfiguration oder wende dich an das NOC.

Nachdem das Kernel-Modul installiert wurde, werden wir :code:`batctl` installieren.

::

   cd /root
   wget https://downloads.open-mesh.org/batman/releases/batman-adv-2018.4/batctl-2018.4.tar.gz
   tar xvf batctl-2018.4.tar.gz
   cd batctl-2018.4/
   make
   make install

Hier könntest du einmal einen Reboot machen. Wenn alles geklappt hat, kannst du nach dem Login als super-user :code:`batctl -v` eingeben.

::

   batctl 2018.4 [batman-adv: 2018.4]

Wenn du hier eine andere Ausgabe bekommst, wendest du dich am besten an das NOC.

fastd
-----

fastd ist die VPN-Software mit der wir die Gateways untereinander und mit den Knoten verbinden. Es handelt sich dabei um einen Layer2 Tunnel. fastd zeichnet sich durch eine relativ hohe effiziens aus und kann daher auch auf den relativ schwachen Knoten laufen. Allerdings belastet die Verschlüsselung der Tunnel nicht nur die Gateways sondern auch die Knoten. Ein Nachteil von fastd sind die häufigen Kontext wechsel zwischen Kernel- und Userland.

Wir beginnen mit der Installation, die aktuelle version ist in den Debian-Paketen enthalten.

::

   apt install fastd

Wir brauchen für den neuen Server die Schlüssel für fastd. Diese sind in
Südholstein für 12 Gateways bereits in der Firmware eingetragen und den
privaten Schlüssel gibt es über das NOC-Team (noc@freifunk-suedholstein.de).

Für fastd brauchen wir ein Schlüssel-Paar einen public und private Key.

Wenn du eines der Gateways aus der Tabelle gewählt hast, dann bekommst du den privaten Schlüssel vom NOC.
Ist dies nicht der Fall, dann musst du selber ein neues Schlüsselpaar erzeugen.

::

  fastd --generate-key

Die Schlüssel sicher Verwahren. Der public Key muss natürlich mit in die Firmware Konfiguration auch die anderen Gateways brauchen eine peer Konfiguration. Wie diese aussieht kannst du auf `GitHub <https://github.com/ffsh/Gateways>`_ sehen.

Jetzt konfigurieren wir Fastd bevor es weiter geht noch einmal die Tabelle.

================== ======= ==========
Datum              Kürzel  Kommentar
------------------ ------- ----------
fastd-MAC          $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC         $MACb   Die MAC-Adresse für das Batman Inteface
IPv4               $IPv4   Feste IPv4 für Freifunk
IPv6               $IPv6   Feste IPV6 für Freifunk
Gateway-Name       $gwName Name deines Gateways
Public-IPv4        $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6        $IPv6P  Öffentliche feste IPv6 deines Gateways
Internet-Interface $InetI  Das Internet-Interface deines Servers
Exit-Interface     $ExitI  Das Exit-Interface
Fastd-Private      $FastdP Fastd Private Key
Fastd-Public       $Fastd  Fastd Public Key
================== ======= ==========

Konfiguration
~~~~~~~~~~~~~

Als erstes erzeugen wir ein neues Verzeichnis, in dem später die Konfiguration abgelegt wird.

::

   mkdir /etc/fastd/ffsh/



In diesem Verzeichnis legen wir eine Konfigurationsdatei für fastd an.
Die Konfigurationsdatei :code:`/etc/fastd/ffsh/fastd.conf` soll diese Zeilen
enthalten:

::


   # Lausche auf jeder IP an Port 10000 auf dem Interface $Exit
   bind any:10000 interface "$InetI";

   # Fastd soll unter dem Benutzer ffsh laufen
   user "ffsh";

   # Hier setzen wir den Namen für das fastd Interface
   interface "ffsh-mesh";

   # Wir benutzen tap
   mode tap;

   # Die MTU ist bei uns 1426 diese muss zu der MTU in der Firmware passen.
   mtu 1426;

   # Hier sind die Methoden für die Verschleierung "null"->Unverschlüsselt die Reihenfolge bestimmt die Priorität.
   method "null";
   method "salsa2012+umac";

   # Der private Key
   secret "$FastdP";

   # Hier das Log setting es wird nach syslog geschrieben man kann bei Problemen auf von info auf debug umstellen.
   log to syslog level info;

   # Hier geben wir ein Verzeichnis an wo die anderen Gateways hinterlegt sind.
   include peers from "gateways/";

   # Dieses Einstellung legt fest, dass unbekannte Clients reingelassen werden.
   on verify "true";

   # Es gibt die Möglichkeit mit einer Blacklist zu arbeiten
   # on verify "/etc/fastd/fastd-blacklist.sh $PEER_KEY";

   # Hier sind einige Befehle die ein paar Wichtige dinge machen
   on up "
    ip link set dev $INTERFACE address $MACf # MAC für fastd-Interface festlegen.
    ip link set dev $INTERFACE up            # Interface auf aktiv stellen.
    ifup bat0                                # Batman-Interface aktiv stellen
    ifup br-ffsh                             # Freifunk-Brücke aktiv stellen
    sh /etc/fastd/ffsh/routing.sh            # Firewall und Routing aktivieren
   ";

   on down "
    ip link set dev $INTERFACE down            # Interface auf inaktiv stellen.
    ifdown bat0                                # Batman-Interface inaktiv stellen
    ifdown br-ffsh                             # Freifunk-Brücke inaktiv stellen
    sh /etc/fastd/ffsh/DROP-routing.sh         # Firewall und Routing deaktivieren
   ";

Ersetze hier: :code:`$InetI, $FastdP, $MACf`.

Datei speichern und fertig. Hier die neue Tabelle:

================== ======= ==========
Datum              Kürzel  Kommentar
------------------ ------- ----------
fastd-MAC          $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC         $MACb   Die MAC-Adresse für das Batman Inteface
IPv4               $IPv4   Feste IPv4 für Freifunk
IPv6               $IPv6   Feste IPV6 für Freifunk
Gateway-Name       $gwName Name deines Gateways
Public-IPv4        $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6        $IPv6P  Öffentliche feste IPv6 deines Gateways
Internet-Interface $InetI  Das Internet-Interface deines Servers
Exit-Interface     $ExitI  Das Exit-Interface
Fastd-Private      $FastdP Fastd Private Key
Fastd-Public       $Fastd  Fastd Public Key
Fastd-Interface    $FastdI Fastd Interface
================== ======= ==========


Netzwerk
--------

Als erstes richten wir IP-Forwarding ein, damit IP-Pakete an andere Geräte weitergeleitet werden.
In der Konfigurationsdatei :code:`/etc/sysctl.d/forwarding.conf`:
::

   # IPv4 Forwarding
   net.ipv4.ip_forward=1

   # IPv6 Forwarding
   net.ipv6.conf.all.forwarding = 1

Danach die Systemweiten Konfigurationsdateien neu laden, damit die Änderung wirksam wird.

::

   sysctl --system

Interfaces
~~~~~~~~~~

Als nächstes werden die Netzwerkschnittstellen :code:`bat0` und :code:`br-ffsh` eingerichtet.

In :code:`/etc/network/interfaces` fügen wir die folgenden Zeilen ein,
dabei musst du ein paar Stellen anpassen. Die Adressen für das Freifunk-Netzwerk kannst du der Tabelle in :ref:`infrastruktur-gateways` entnehmen.

::


   #
   # Netwerkbrücke für Freifunk
   # Hier läuft der Traffic von den einzelnen Routern und dem externen VPN zusammen.
   # Unter der hier konfigurierten IP ist der Server selber im Freifunk Netz erreichbar.
   # bridge_ports none sorgt dafür, dass die Brücke auch ohne Interface erstellt wird.
   #

   allow-hotplug br-ffsh
   iface br-ffsh inet static
       address $IPv4
       netmask 16
       bridge_ports none

   iface br-ffsh inet6 static
       address $IPv6
       netmask 64

   #
   # Batman Interface
   # Erstellt das virtuelle Inteface für das Batman-Modul und bindet dieses an die Netzwerkbrücke
   #

   allow-hotplug bat0
   iface bat0 inet6 manual
       pre-up batctl if add ffsh-mesh
       post-up ip link set address $MACb dev bat0

       post-up brctl addif br-ffsh bat0
       post-up batctl it 10000
       post-up batctl gw server 100mbit/100mbit

       pre-down brctl delif br-ffsh bat0 || true

Ersetze hier: :code:`$IPv4, $IPv6, $MACb`.

FQDN, Hosts
~~~~~~~~~~~

Achtung! Vermutlich enthält deine :code:`/etc/hosts` bereits Einträge für diene öffentlichen Adressen, füge dort nur den FQDN und deinen Gateway-Namen hinzu.

::


   127.0.0.1                  localhost
   $IPv4P                     $gwName.freifunk-suedholstein.de $gwName
   $IPv6P                     $gwName.freifunk-suedholstein.de $gwName
   $IPv4                      $gwName.freifunk-suedholstein.de $gwName
   $IPv6                      $gwName.freifunk-suedholstein.de $gwName

iptables: Defaults
~~~~~~~~~~~~~~~~~~

Lege die Konfigurationsdatei :code:`/etc/iptables.up.rules` an mit Folgendem:

::

   *filter
   :INPUT ACCEPT [0:0]
   :FORWARD ACCEPT [0:0]
   :OUTPUT ACCEPT [0:0]
   COMMIT
   *mangle
   :PREROUTING ACCEPT [0:0]
   :INPUT ACCEPT [0:0]
   :FORWARD ACCEPT [0:0]
   :OUTPUT ACCEPT [0:0]
   :POSTROUTING ACCEPT [0:0]
   COMMIT
   *nat
   :PREROUTING ACCEPT [0:0]
   :INPUT ACCEPT [0:0]
   :OUTPUT ACCEPT [0:0]
   :POSTROUTING ACCEPT [0:0]
   COMMIT

Damit wurden nun Default-Policys festgelegt.

Nun müssen die IP-Tables geladen werden. Bitte erstellt die Datei
:code:`/etc/network/if-pre-up.d/iptables` mit folgenden Zeilen:

::

   #!/bin/sh
   /sbin/iptables-restore < /etc/iptables.up.rules

Zum Schluss laden wir die Default-Policys:

::

   chmod 700 /etc/network/if-pre-up.d/iptables
   iptables-restore < /etc/iptables.up.rules

Routing: fastd
~~~~~~~~~~~~~~

Für das Routing brauchen wir ein paar Routing- und Firewall-Regeln diese werden in :code:`/etc/fastd/ffsh/routing.sh` gespeichert.
Dieses Skript wird durch fastd gestartet.

::

   #!/bin/sh

   #
   # Routing table
   #

   # Route zurück ins Freifunk Netzwerk
   /sbin/ip route add table ffsh 10.144.0.0/16 dev br-ffsh src $IPv4

   # Temporäre blockaden
   /sbin/ip route add table ffsh 128/1 reject
   /sbin/ip route add table ffsh 0/1 reject

   # Alle Pakete von interface "br-ffsh" über Tabelle ffsh
   /sbin/ip rule add table ffsh iif br-ffsh
   # Alle Pakete mit der Markierung 0x1 über Tabelle ffsh
   /sbin/ip rule add table ffsh from all fwmark 0x1

   #
   # Firewall Rules
   #

   # Alle Pakete aus Freifunk mit 0x1 markieren
   /sbin/iptables -t mangle -I PREROUTING -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # Traffic vom Gateway Richtung Freifunk markieren mit 0x1
   /sbin/iptables -t mangle -I OUTPUT -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # NAT
   /sbin/iptables -t nat -I POSTROUTING -s 0/0 -d 0/0 -j MASQUERADE

Ersetze hier: :code:`$IPv4`.

Um die Regeln beim stoppen von fastd rückgängig zu machen legen wir eine zweite Datei an. :code:`/etc/fastd/ffsh/DROP-routing.sh`

::

   #!/bin/sh

   #
   # DROP Routing table
   #

   # Route zurück ins Freifunk Netzwerk
   /sbin/ip route del table ffsh 10.144.0.0/16 dev br-ffsh src $IPv4

   # Temporäre blockaden
   /sbin/ip route del table ffsh 128/1 reject
   /sbin/ip route del table ffsh 0/1 reject

   # Alle Pakete von interface "br-ffsh" über Tabelle ffsh
   /sbin/ip rule del table ffsh iif br-ffsh
   # Alle Pakete mit der Markierung 0x1 über Tabelle ffsh
   /sbin/ip rule del table ffsh from all fwmark 0x1

   #
   # DROP Firewall Rules
   #

   # Alle Pakete aus Freifunk mit 0x1 markieren
   /sbin/iptables -t mangle -D PREROUTING -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # Traffic vom Gateway Richtung Freifunk markieren mit 0x1
   /sbin/iptables -t mangle -D OUTPUT -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # NAT
   /sbin/iptables -t nat -D POSTROUTING -s 0/0 -d 0/0 -j MASQUERADE

Ersetze hier: :code:`$IPv4`.

Jetzt machen wir die beiden Dateien ausführbar.

::

   chmod 700 /etc/fastd/ffsh/routing.sh
   chmod 700 /etc/fastd/ffsh/routing-DROP.sh


Routing: Exit
~~~~~~~~~~~~~

Die Regeln aus dem vorherigen Abschnitt ermöglicht noch keinen Weg ins Internet. Wir hinterlegen die Regeln hierfür in einem zweiten Skript, um eine möglichst hohe Modularität zu erreichen.

Das folgende Skript soll die Regeln, welche durch fastd gesetzt wurden durch Routen ins Internet ersetzen. Außerdem werden einige Filterregeln installiert. Diese Verhindern das scannen von nicht im Internet vorhanden Adressen und verhindern insbesondere bei einem RAW-Exit Ärger mit dem Provider.

:code:`/home/ffsh/exit.sh`

::

   #!/bin/sh

   #
   # Routing
   #

   /sbin/ip route replace table ffsh 128/1 dev $ExitI
   /sbin/ip route replace table ffsh 0/1 dev $ExitI

   #
   # Optional: Filter
   #

   /sbin/iptables -A OUTPUT -o $ExitI -d 169.254.0.0/16 -j DROP
   /sbin/iptables -A OUTPUT -o $ExitI -d 172.16.0.0/12 -j DROP
   /sbin/iptables -A OUTPUT -o $ExitI -d 127.0.0.0/8 -j DROP
   /sbin/iptables -A OUTPUT -o $ExitI -d 10.0.0.0/8 -j DROP
   /sbin/iptables -A OUTPUT -o $ExitI -d 192.168.0.0/16 -j DROP

   /sbin/iptables -A FORWARD -d 169.254.0.0/16 -j DROP
   /sbin/iptables -A FORWARD -d 172.16.0.0/12 -j DROP
   /sbin/iptables -A FORWARD -d 127.0.0.0/8 -j DROP
   /sbin/iptables -A FORWARD -d 192.168.0.0/16 -j DROP

   /sbin/iptables -A FORWARD -o $ExitI -d 10.0.0.0/8 -j DROP

   /sbin/ip6tables -A FORWARD -p tcp --dport 7 -j DROP
   /sbin/iptables -A FORWARD -p tcp --dport 68 -j DROP
   /sbin/ip6tables -A FORWARD -p tcp --dport 546 -j DROP

Ersetze hier :code:`$ExitI`.


Auch für dieses Skript legen wir Regeln zum rückgängig machen fest. :code:`/home/ffsh/DROP-exit.sh`

::

   #
   # Routing
   #

   /sbin/ip route replace table ffsh 128/1 reject
   /sbin/ip route replace table ffsh 0/1 reject

   #!/bin/sh

   #
   # DROP Filter
   #

   /sbin/iptables -D OUTPUT -o $ExitI -d 169.254.0.0/16 -j DROP
   /sbin/iptables -D OUTPUT -o $ExitI -d 172.16.0.0/12 -j DROP
   /sbin/iptables -D OUTPUT -o $ExitI -d 127.0.0.0/8 -j DROP
   /sbin/iptables -D OUTPUT -o $ExitI -d 10.0.0.0/8 -j DROP
   /sbin/iptables -D OUTPUT -o $ExitI -d 192.168.0.0/16 -j DROP

   /sbin/iptables -D FORWARD -d 169.254.0.0/16 -j DROP
   /sbin/iptables -D FORWARD -d 172.16.0.0/12 -j DROP
   /sbin/iptables -D FORWARD -d 127.0.0.0/8 -j DROP
   /sbin/iptables -D FORWARD -d 192.168.0.0/16 -j DROP

   /sbin/iptables -D FORWARD -o $ExitI -d 10.0.0.0/8 -j DROP

   /sbin/ip6tables -D FORWARD -p tcp --dport 7 -j DROP
   /sbin/iptables -D FORWARD -p tcp --dport 68 -j DROP
   /sbin/ip6tables -D FORWARD -p tcp --dport 546 -j DROP

Ersetze hier :code:`$ExitI`.


Die Skripte machen wir nun ausführbar:

::

   chmod 700 /home/ffsh/exit.sh
   chmod 700 /home/ffsh/DROP-exit.sh


Dienste
-------

Im folgenden werden einige essentielle Dienste eingerichtet.

radvd
~~~~~

TODO

::

   apt install radvd

Es wird für IPv6 die Konfigurationsdatei :code:`/etc/radvd.conf` mit folgenden
Zeilen benötigt:

::

   interface br-ffsh {
       AdvSendAdvert on;
       IgnoreIfMissing on;
       AdvManagedFlag off;
       AdvOtherConfigFlag on;
       MaxRtrAdvInterval 200;
       AdvLinkMTU 1280;
       prefix fddf:0bf7:80::/64 {
           AdvOnLink on;
           AdvAutonomous on;
           AdvRouterAddr on;
       };

       RDNSS $IPv6 {
       };
   };

Ersetze hier :code:`IPv6`.

Zum Abchluss starten wir radvd neu.

::

   systemctl restart radvd

dhcp
~~~~
TODO

::

   apt install isc-dhcp-server


Die Konfigurationsdatei :code:`/etc/dhcp/dhcpd.conf` wird für IPv4 mit folgenden
Zeilen benötigt:

::

   ddns-update-style none;
   option domain-name ".ffsh";

   # möglichst kurze Leasetime
   default-lease-time 120;
   max-lease-time 600;

   log-facility local7;

   subnet 10.144.0.0 netmask 255.255.0.0 {
       authoritative;
       range 10.144.[GW Netz].2 10.144.[GW Netz + 15].254;

       option routers $IPv4;

       option domain-name-servers $IPv4;
   }

   include "/etc/dhcp/static.conf";

Ersetze hier :code:`$IPv4`

Bitte eine leere Datei :code:`/etc/dhcp/static.conf` erzeugen.

::

    useradd -m -s /bin/bash dhcpstatic

    cd /home/dhcpstatic

    su dhcpstatic

    git clone https://github.com/ffsh/dhcp-static.git

    chmod +x dhcp-static/updateStatics.sh

    exit

    /home/dhcpstatic/dhcp-static/updateStatics.sh

    */5 * * * * root /home/dhcpstatic/dhcp-static/updateStatics.sh > /dev/null 2>&1

Auf dem DHCP-Server muss noch das Bridge-Interface für IPv4 festgelegt
werden. Bitte die Datei :code:`/etc/default/isc-dhcp-server` mit folgender
Option ergänzen:

::

   # On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
   # Separate multiple interfaces with spaces, e.g. "eth0 eth1".
   INTERFACES="br-ffsh"

Am Besten wird der DHCP-Server vor dem Start und Betrieb noch mal
geprüft. Bitte vorher den Server rebooten und dann auf der Konsole als
root folgende Zeile ausführen:

::

   dhcpd -f -d


War das erfolgreich, so kann der DHCP-Server als root gestartet werden:

::

   systemctl restart isc-dhcp-server


DNS-Server (BIND)
~~~~~~~~~~~~~~~~~

::

   apt install bind9


Für das interne Freifunknetz ist nun noch der DNS-Server bind9 mit den
Konfigurationsdateien wie folgt zu konfigurieren:

Erstmal diese Datei :code:`/etc/bind/named.conf.options`

::


   options {
       directory "/var/cache/bind";
       // If there is a firewall between you and nameservers you want
       // to talk to, you may need to fix the firewall to allow multiple
       // ports to talk.  See http://www.kb.cert.org/vuls/id/800113
       // If your ISP provided one or more IP addresses for stable
       // nameservers, you probably want to use them as forwarders.
       // Uncomment the following block, and insert the addresses replacing
       // the all-0's placeholder.
       forwarders {
           8.8.8.8;
           8.8.4.4;
       };
       //========================================================================
       // If BIND logs error messages about the root key being expired,
       // you will need to update your keys.  See https://www.isc.org/bind-keys
       //========================================================================
       // dnssec-enable yes;
       // dnssec-validation yes;
       dnssec-validation no;
       // dnssec-lookaside auto;
       // recursion yes;
       // allow-recursion { localnets; localhost; };
       auth-nxdomain no;    # conform to RFC1035
       listen-on-v6 { any; };
   };


Dann in der Datei :code:`/etc/bind/named.conf.local` folgendes am Ende ergänzen:

::


   // Do any local configuration here
   // Consider adding the 1918 zones here, if they are not used in your organization

   include "/etc/bind/zones.rfc1918";

   zone "stormarn.freifunk.net" {
          type master;
          file "/etc/bind/db.net.freifunk.stormarn";
     };

   zone "freifunk-stormarn.de" {
          type master;
          file "/etc/bind/db.de.freifunk-stormarn";
     };

   zone "lauenburg.freifunk.net" {
          type master;
          file "/etc/bind/db.net.freifunk.lauenburg";
     };

   zone "freifunk-lauenburg.de" {
          type master;
          file "/etc/bind/db.de.freifunk-lauenburg";
     };

   zone "freifunk-suedholstein.de" {
           type master;
           file "/etc/bind/db.de.freifunk-suedholstein";
   };

   zone "ffshev.de" {
        type master;
        file "/etc/bind/db.de.ffshev";
   };



Die zugehörigen Zone Dateien werden in einem
`Repository <https://github.com/ffsh/bind>`_ verwaltet.

Diese sollen automatisch aktualisiert werden.

Als erstes legen wir einen neuen Benutzer an.

::

    adduser --disabled-login dnsbind

Dann wechseln wir zu diesem Nutzer.

::

    su - dnsbind
    cd /home/dnsbind/

Und Klonen das Repository

::

    git clone https://github.com/ffsh/bind.git

Danach verlassen wir den Nutzer.

::

    exit

Und legen einige Cron jobs an.

::

    */15 * * * * root /home/dnsbind/bind/updatestofrei.sh > /dev/null 2>&1
    */15 * * * * root /home/dnsbind/bind/updatelauen.sh > /dev/null 2>&1
    */15 * * * * root /home/dnsbind/bind/updateffsh.sh > /dev/null 2>&1

Zum Schluss starten wir bind neu.

::


   systemctl restart bind9


Mesh Announce
~~~~~~~~~~~~~

Um als Gateway, Server oder alles was kein Freifunk Router ist auf der
Karte zu erscheinen kann `Mesh Announce <https://github.com/ffnord/mesh-announce>`_ installiert
werden.

Dafür müssen folgende Dinge vorhanden sein:

::


   lsb_release, ethtool, python3 (>= 3.3)
   sudo apt install ethtool python3


Mesh Announce kann auch im alfred Stil Daten broadcasten das wollen wir
aber nicht.

::


   sudo git clone https://github.com/ffnord/mesh-announce /opt/mesh-announce
   sudo cp /opt/mesh-announce/respondd.service /etc/systemd/system/respondd.service
   nano /etc/systemd/system/respondd.service


Den Systemd Service passen wir jetzt an unser Netzwerk und Gateway an. Erstmal das Konzept. Wir starten respondd.py mit einigen argumenten:

::

   respondd.py -d /opt/mesh-announce/providers -i <your-clientbridge-if> -i <your-mesh-vpn-if> -b <your-batman-if> -m <mesh ipv4 address>

::


   your-clientbridge-if - br-ffsh
   your-mesh-vpn-if     - ffsh-mesh
   your-batman-if       - bat0
   mesh ipv4 address    - GW-IPV4

Im folgenden Beispiel ist Hopfenbach das Gateway dort sind die Interfaces so wie in der Anleitung benannt und die IP ist :code:`10.144.128.1`.
::


   [Unit]
   Description=Respondd
   After=network.target

   [Service]
   ExecStart=/opt/mesh-announce/respondd.py -d /opt/mesh-announce/providers -i br-ffsh -i ffsh-mesh -b bat0 -m 10.144.128.1
   Restart=always
   Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

   [Install]
   WantedBy=multi-user.target



Dann mit :code:`hostname` prüfen ob der erwünschte Gateway-Name eingetragen ist
ggf. ändern oder:

:code:`nano providers/nodeinfo/hostname.py`

::


  import providers
  import socket
  class Source(providers.DataSource):
      def call(self):
          return "GW_Barnitz"



Dann den Service aktivieren

::


   systemctl daemon-reload
   systemctl start respondd
   # autostart on boot
   systemctl enable respondd


Das System sollte in kürze auf der Karte auftauchen.

Für VPN: Regelmäßig testen
--------------------------

Wenn möglich, dann ist es sinnvoll den VPN-Ausgang regelmäßig zu testen.
Wenn du auf dem Interface einen Ping an google senden kannst dann eignet sich dieses Skript:

:code:`/root/check-vpn.sh`

::

   #!/bin/bash

   # Test gateway is connected to VPN
   test=$(ping -q -I $Exit 8.8.8.8 -c 4 -i 1 -W 5 | grep 100 )

   if [ "$test" != "" ]
       then
       echo "VPN nicht da - Neustart!"
       sytemctl restart openvpn       # Fehler - VPN nicht da - Neustart
   else
       echo "alles gut"
   fi

Ersetze hier :code:`$Exit` durch das VPN-Interface

Dann noch das Script ausführbar machen:

::

   chmod u+x /root/check-vpn.sh


Danach in die Datei :code:`/etc/crontab` das Skript alle 10 Minute auszuführen
und damit regelmäßig der VPN-Status geprüft wird.

::

   # Check VPN via openvpn is running, if not service restart
   */10 * * * * root /root/check-vpn.sh > /dev/null

Die Änderungen übernehmen durch einen Neustart des Cron-Dämonen:

::

   systemctl restart cron

Wenn du alles richtig gemacht hast und dein Exit-Interface funktioniert dann gehen wir über zum nächsten Schritt.