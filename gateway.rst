.. toctree::
   :maxdepth: 2
   :caption: Inhalte

Gateway Konfiguration
=====================

In diesem Kapitel konfigurieren wir das Gateway dabei gehen wir Schritt für Schritt vor.

Wenn du ein neues Gateway einrichten willst solltest du dir als erstes ein freies aus der Tabelle :ref:`infrastruktur-gateways` aussuchen.
Wenn du keines der bereits definierten Gateways benutzen möchtest dann brauchst du folgende Daten:

=========== ======= ==========
Datum       Kürzel  Kommentar
----------- ------- ----------
fastd-MAC   $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC  $MACb   Die MAC-Adresse für das Batman Inteface
IPv4        $IPv4   Feste IPv4 für Freifunk
IPv6        $IPv6   Feste IPV6 für Freifunk
Public-IPv4 $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6 $IPv6P  Öffentliche feste IPv6 deines Gateways
=========== ======= ==========

Zu der :code:`$IPv4` und :code:`$IPv6` gehören natürlich noch jeweils das Passende Netzwerk, welches an die Clients verteilt wird. Wähle also passende Subnetze aus und dazu gehörende Adressen für dein Gateway.

Als nächstes solltest du dir Hardware oder ein VM suchen auf der dein Gateway laufen soll. Dabei gibt es die wichtige Voraussetzung, dass du Kernel-Module installieren kannst dabei kommt es auf die eingesetzte Virtualisierungstechnik an. Wenn der Anbieter kvm benutzt ist alles gut. Wir haben gute Erfahrungen mit der Hetzner Cloud gemacht. Nach dem du dir dein System ausgesucht hast solltest du dir ein Betriebssystem aussuchen wir nehmen Debian (9).

Wenn du den Traffic der Clients nicht direkt an deinem Gateway ins Internet lassen möchtest, dann solltest du dir auch einen VPN-Anbieter suchen. Wir haben gute Erfahrungen mit Mullvad und Private Internet Access gemacht aber es sollten auch andere VPN-Anbieter funktionieren.
Nach der Standard Debian Installation (minimal reicht). Verbindest du dich mit deinem Gateway. Und wechselst in den Super-User :code:`sudo su`.

Allgemeine Software Pakete
--------------------------

Zum Einstieg installieren wir einige Pakete die wir später brauchen werden.

::

   sudo apt install build-essential git apt-transport-https bridge-utils ntp net-tools

VPN oder Direkt
---------------

Spätestens jetzt solltest du dich entschieden haben ob du einen VPN-Anbieter verwenden möchtest oder nicht.
Die Meisten Anbieter werden eine Anleitung für Debian haben, wenn nicht solltest du vielleicht einen anderen wählen.

Für die Freifunk-Installation interessiert uns am Ende der Name des Interfaces.

Wenn du direkt ausleiten möchtest dann brauchen wir den Namen deines "Internet-Interfaces".

Egal für welche Variante du dich entscheidest wir merken uns den Interface Namen. Zu der Tabelle am Anfang kommt also ein neuer Eintrag.

============== ======= ==========
Datum          Kürzel  Kommentar
-------------- ------- ----------
fastd-MAC      $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC     $MACb   Die MAC-Adresse für das Batman Inteface
IPv4           $IPv4   Feste IPv4 für Freifunk
IPv6           $IPv6   Feste IPV6 für Freifunk
Public-IPv4    $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6    $IPv6P  Öffentliche feste IPv6 deines Gateways
Exit-Interface $Exit   Das Exit Interface z.b. tun0 oder eth1
============== ======= ==========


Für VPN: Regelmäßig testen
~~~~~~~~~~~~~~~~~~~~~~~~~~

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

Batman und Fastd
----------------

Freifunk Südholstein Benutzt für das Mesh-Netzwerk batman-adv und den Routing-Algorithmus Batman IV. Für die VPN-Tunnel zu den Knoten benutzen wir Fastd.
Da das Batman-Kernel-Modul nicht in einer aktuellen Version im OS-Repository ist, müssen wir es selber bauen.

Batman Kernel-Modul und batctl
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Für Batman gibt es ein Kernel-Modul und ein Kontroll-Programm. Als erstes installieren wir aber ein paar Pakete die wir benötigen.

::

    apt install linux-headers-amd64

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

danach prüfen wir ob alles funktioniert.
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

fastd v18 ist in Debian 9 bereits in den Repositories enthalten. Unter
Debian 8 findet man es in den jessie-backports.

::

   apt install fastd

fastd-Konfiguration
~~~~~~~~~~~~~~~~~~~

Wir brauchen für den neuen Server die Schlüssel für fastd. Diese sind in
Südholstein für 12 Gateways bereits in der Firmware eingetragen und den
privaten Schlüssel gibt es über das NOC-Team (noc@freifunk-suedholstein.de).

Für fastd brauchen wir ein Schlüssel-Paar einen public und private Key.

Wenn du eines der Gateways aus der Tabelle gewählt hast, dann bekommst du den privaten Schlüssel vom NOC.
Ist dies nicht der Fall, dann musst du selber ein neues Schlüsselpaar erzeugen.

::

  fastd --generate-key

Die Schlüssel sicher Verwahren. Der public Key muss später in der Firmware eingetragen werden.

Jetzt konfigurieren wir Fastd bevor es weiter geht noch einmal die Tabelle.

============== ======= ==========
Datum          Kürzel  Kommentar
-------------- ------- ----------
fastd-MAC      $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC     $MACb   Die MAC-Adresse für das Batman Inteface
IPv4           $IPv4   Feste IPv4 für Freifunk
IPv6           $IPv6   Feste IPV6 für Freifunk
Public-IPv4    $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6    $IPv6P  Öffentliche feste IPv6 deines Gateways
Exit-Interface $Exit   Das Exit Interface z.b. tun0 oder eth1
Fastd-Private  $FastdP Fastd Private Key
Fastd-Public   $Fastd  Fastd Public Key
============== ======= ==========

Als erstes erzeugen wir ein neues Verzeichnis, in dem später die Konfiguration abgelegt wird.

::

   mkdir /etc/fastd/ffsh/

Außerdem benötigen wir einen Nutzer ohne root-Rechte, unter dem fastd laufen kann.

::

   adduser --disabled-login --no-create-home ffsh

Die darauf folgenden Fragen nach Namen, Orten und Firmenzugehörigkeit können alle leer gelassen werden.

In diesem Verzeichnis legen wir eine Konfigurationsdatei für fastd an.
Die Konfigurationsdatei :code:`/etc/fastd/ffsh/fastd.conf` soll diese Zeilen
enthalten:

::


   # Lausche auf jeder IP an Port 10000 auf dem Interface $Exit
   bind any:10000 interface "$Exit";

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
    # sh /etc/fastd/ffsh/filtering.sh        # Optional Filterregeln später mehr dazu
   ";

   # Hier wird alles rückgängig gemacht.
   on down "
    ifdown bat0
    ifdown br-ffsh
    sh /etc/fastd/ffsh/DROP-routing.sh
    # sh /etc/fastd/ffsh/DROP-filtering.sh
   ";

Datei speichern und fertig. Hier die neue Tabelle:

=============== ======= ==========
Datum           Kürzel  Kommentar
--------------- ------- ----------
fastd-MAC       $MACf   Die MAC-Adresse für den Fastd Tunnel
batman-MAC      $MACb   Die MAC-Adresse für das Batman Inteface
IPv4            $IPv4   Feste IPv4 für Freifunk
IPv6            $IPv6   Feste IPV6 für Freifunk
Public-IPv4     $IPv4P  Öffentliche feste IPv4 deines Gateways
Public-IPv6     $IPv6P  Öffentliche feste IPv6 deines Gateways
Exit-Interface  $Exit   Das Exit Interface z.b. tun0 oder eth1
Fastd-Private   $FastdP Fastd Private Key
Fastd-Public    $Fastd  Fastd Public Key
Fastd-Interface $FastdI Fastd Interface
=============== ======= ==========

Netzwerk Konfiguration
----------------------

Im Folgenden werden wir das Routing und die Netzwerkschnittstellen auf dem Gateway einrichten.
Du solltest die IP-Adressen deines Gateways bereit halten.

IP Forwarding
~~~~~~~~~~~~~

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

Interfaces Konfigurieren
~~~~~~~~~~~~~~~~~~~~~~~~

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
       address 10.144.[GW Netz].1 # Deine Gateway IPv4-Adresse
       netmask 255.255.0.0
       bridge_ports none

   iface br-ffsh inet6 static
       address fddf:0bf7:80::[GW Netz]:1 # Deine Gateway IPv6-Adresse
       netmask 64
   #
   # Batman Interface
   # Erstellt das virtuelle Inteface für das Batman-Modul und bindet dieses an die Netzwerkbrücke
   #

   allow-hotplug bat0
   iface bat0 inet6 manual
       pre-up batctl if add ffsh-mesh
       post-up ip link set address 00:5b:27:81:0[GW Netz] dev bat0   # ACHTUNG BEI GW NETZ DEN DOPPELPUNKT NICHT VERGESSEN (80=0:80 128=1:28)

       post-up brctl addif br-ffsh bat0
       post-up batctl it 10000
       post-up batctl gw server 100mbit/100mbit

       pre-down brctl delif br-ffsh bat0 || true

Die :code:`/etc/hosts` mit Folgenden Zeilen befüllen:

::


   127.0.0.1                  localhost
   [externe IP]               [GW Name].freifunk-suedholstein.de   [GW Name]
   10.144.[GW Netz].1         ffsh
   fddf:0bf7:80::[GW Netz]:1  ffsh


IP Tables
~~~~~~~~~

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


Für das Routing brauchen wir ein paar Routing- und Firewall-Regeln diese werden in :code:`/etc/fastd/ffsh/routing.sh` gespeichert.

::

   #!/bin/sh

   #
   # Routing table
   #

   # Route zurück ins Freifunk Netzwerk
   /sbin/ip route add table 42 10.144.0.0/16 dev br-ffsh src 10.144.[$GW].1
   # Route nach link-local
   /sbin/ip route add table 42 128/1 dev tun0
   # Route für den Rest, Internet und so
   /sbin/ip route add table 42 0/1 dev tun0

   # Alle Pakete von interface "br-ffsh" über Tabelle 42
   /sbin/ip rule add table 42 iif br-ffsh
   # Alle Pakete mit der Markierung 0x1 über Tabelle 42
   /sbin/ip rule add table 42 from all fwmark 0x1

   #
   # Firewall Rules
   #

   # Alle Pakete aus Freifunk mit 0x1 markieren
   /sbin/iptables -t mangle -I PREROUTING -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # Traffic vom Gateway Richtung Freifunk markieren mit 0x1
   /sbin/iptables -t mangle -I OUTPUT -s 10.144.0.0/16 -j MARK --set-mark 0x1
   # NAT
   /sbin/iptables -t nat -I POSTROUTING -s 0/0 -d 0/0 -j MASQUERADE

Besonders bei direkter Ausleitung des Traffics ist es wichtig IP-Bereiche, die im Internet nicht gerouted werden können, zu filtern. Auf Gateways mit VPN Tunnel tut es aber nicht weh, hier ist das exit-interface der VPN-Tunnel.

:code:`/etc/fastd/ffsh/filtering.sh`

::

   #!/bin/sh

   #
   # Filter
   #
   /sbin/iptables -A OUTPUT -o [EXIT-Interface] -d 169.254.0.0/16 -j DROP
   /sbin/iptables -A OUTPUT -o [EXIT-Interface] -d 172.16.0.0/12 -j DROP
   /sbin/iptables -A OUTPUT -o [EXIT-Interface] -d 127.0.0.0/8 -j DROP
   /sbin/iptables -A OUTPUT -o [EXIT-Interface] -d 10.0.0.0/8 -j DROP
   /sbin/iptables -A OUTPUT -o [EXIT-Interface] -d 192.168.0.0/16 -j DROP

   /sbin/iptables -A FORWARD -d 169.254.0.0/16 -j DROP
   /sbin/iptables -A FORWARD -d 172.16.0.0/12 -j DROP
   /sbin/iptables -A FORWARD -d 127.0.0.0/8 -j DROP
   /sbin/iptables -A FORWARD -d 192.168.0.0/16 -j DROP

   /sbin/iptables -A FORWARD -o [EXIT-Interface] -d 10.0.0.0/8 -j DROP

   /sbin/ip6tables -A FORWARD -p tcp --dport 7 -j DROP
   /sbin/iptables -A FORWARD -p tcp --dport 68 -j DROP
   /sbin/ip6tables -A FORWARD -p tcp --dport 546 -j DROP


Um die Regeln beim stoppen oder neustarten von fastd zu entfernen legen wir weitere Skripte an:

Für das Routing :code:`/etc/fastd/ffsh/routing-DROP.sh`.

::


   #!/bin/sh

   #
   # DROP Routing-Table
   #

   /sbin/ip route del table 42 10.144.0.0/16 dev br-ffsh src 10.144.[$GW].1
   /sbin/ip route del table 42 128/1 dev tun0
   /sbin/ip route del table 42 0/1 dev tun0

   /sbin/ip rule del table 42 iif br-ffsh
   /sbin/ip rule del table 42 from all fwmark 0x1

   #
   # DROP Firewall-Rules
   #

   /sbin/iptables -t mangle -D PREROUTING -s 10.144.0.0/16 -j MARK --set-mark 0x1
   /sbin/iptables -t mangle -D OUTPUT -s 10.144.0.0/16 -j MARK --set-mark 0x1
   /sbin/iptables -t nat -D POSTROUTING -s 0/0 -d 0/0 -j MASQUERADE

Und für die Filter :code:`/etc/fastd/ffsh/filtering-DROP.sh`

::


   #!/bin/sh

   #
   # DROP Filter
   #

   /sbin/iptables -D OUTPUT -o [EXIT-Interface] -d 169.254.0.0/16 -j DROP
   /sbin/iptables -D OUTPUT -o [EXIT-Interface] -d 172.16.0.0/12 -j DROP
   /sbin/iptables -D OUTPUT -o [EXIT-Interface] -d 127.0.0.0/8 -j DROP
   /sbin/iptables -D OUTPUT -o [EXIT-Interface] -d 10.0.0.0/8 -j DROP
   /sbin/iptables -D OUTPUT -o [EXIT-Interface] -d 192.168.0.0/16 -j DROP

   /sbin/iptables -D FORWARD -d 169.254.0.0/16 -j DROP
   /sbin/iptables -D FORWARD -d 172.16.0.0/12 -j DROP
   /sbin/iptables -D FORWARD -d 127.0.0.0/8 -j DROP
   /sbin/iptables -D FORWARD -d 192.168.0.0/16 -j DROP

   /sbin/iptables -D FORWARD -o [EXIT-Interface] -d 10.0.0.0/8 -j DROP

   /sbin/ip6tables -D FORWARD -p tcp --dport 7 -j DROP
   /sbin/iptables -D FORWARD -p tcp --dport 68 -j DROP
   /sbin/ip6tables -D FORWARD -p tcp --dport 546 -j DROP

Jetzt sollten vier Skripte in dem Verzeichnis liegen:

::


   /etc/fastd/ffsh/routing.sh        # Skript für das Routing
   /etc/fastd/ffsh/filtering.sh      # Skript für filtern privater IP-Räume etc.
   /etc/fastd/ffsh/routing-DROP.sh   # Skript zum entfernen der Routing-Regeln
   /etc/fastd/ffsh/filtering-DROP.sh #Skript zum entfernen der Filter-Regeln

Die Skripte machen wir nun ausführbar:

::


   chmod 700 /etc/fastd/ffsh/routing.sh
   chmod 700 /etc/fastd/ffsh/filtering.sh
   chmod 700 /etc/fastd/ffsh/routing-DROP.sh
   chmod 700 /etc/fastd/ffsh/filtering-DROP.sh


   chmod 700 /etc/network/if-pre-up.d/iptables

Zum Schluss laden wir die Default-Policys:

::


   iptables-restore < /etc/iptables.up.rules

Die anderen Regeln werden über fastd gestartet.


DHCP
----

::

   apt install radvd isc-dhcp-server


DHCP radvd IPv6
~~~~~~~~~~~~~~~

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

       RDNSS fddf:0bf7:80::[GW Netz]:1 {
       };
   };


Jetzt kann radvd als :code:`root` auf der Konsole gestartet werden:

::

   service radvd restart


DHCP isc-dhcp-server IPv4 und IPv6
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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

       option routers 10.144.[GW Netz].1;

       option domain-name-servers 10.144.[GW Netz].1; # für die eigenen DNS-Einträge
       # option domain-name-servers 85.214.20.141; # weitere anonyme DNS
       # option domain-name-servers 213.73.91.35;
   }

   include "/etc/dhcp/static.conf";


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
-----------------

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
`Repository <https://github.com/ffsh/bind>`__ verwaltet.

Diese sollen automatisch aktualisiert werden.

Als erstes legen wir einen neuen Benutzer an.

::

    useradd -m -s /bin/bash dnsbind

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
-------------

Um als Gateway, Server oder alles was kein Freifunk Router ist auf der
Karte zu erscheinen kann
`mesh-announce <https://github.com/ffnord/mesh-announce>`__ installiert
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