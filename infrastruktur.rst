.. toctree::
   :maxdepth: 2
   :caption: Inhalte

Infrastruktur
=============

Netzwerk
--------

Network IPv4: :code:`10.144.0.0/16`

Network IPv6: :code:`fddf:0bf7:80::/48`


Gateways
--------

+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Name       | ULA                 | IPv6 | RFC1918      | DHCP                        | ICVPN-Transit | Mesh MAC(s)       | B.A.T.M.A.N.-adv. MAC(s) | Standort      | Betreuer | Exit/VPN-Dienst    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Barnitz    | fddf:0bf7:80::48:1  | ULA  | 10.144.48.1  | 10.144.48.2-10.144.63.254   | n/a           | 00:5b:27:80:00:48 | 00:5b:27:81:00:48        | Hetzner (nbg) | ul       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Beste      | fddf:0bf7:80::64:1  | ULA  | 10.144.64.1  | 10.144.64.2-10.144.79.254   | n/a           | 00:5b:27:80:00:64 | 00:5b:27:81:00:64        | Hetzner (fsn) | ul       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Bille      | fddf:0bf7:80::80:1  | ULA  | 10.144.80.1  | 10.144.80.2-10.144.95.254   | n/a           | 00:5b:27:80:00:80 | 00:5b:27:81:00:80        | Netcup        | fr       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Brunsbach  | fddf:0bf7:80::96:1  | ULA  | 10.144.96.1  | 10.144.96.2-10.144.111.254  | n/a           | 00:5b:27:80:00:96 | 00:5b:27:81:00:96        | Hetzber (arm) | be       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Heilsau    | fddf:0bf7:80::112:1 | ULA  | 10.144.112.1 | 10.144.112.2-10.144.127.254 | n/a           | 00:5b:27:80:01:12 | 00:5b:27:81:01:12        | Hetzner (hel) | ul       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Hopfenbach | fddf:0bf7:80::128:1 | ULA  | 10.144.128.1 | 10.144.128.2-10.144.143.254 | n/a           | 00:5b:27:80:01:28 | 00:5b:27:81:01:28        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Krummbach  | fddf:0bf7:80::144:1 | ULA  | 10.144.144.1 | 10.144.144.2-10.144.159.254 | n/a           | 00:5b:27:80:01:44 | 00:5b:27:81:01:44        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Piepenbek  | fddf:0bf7:80::160:1 | ULA  | 10.144.160.1 | 10.144.160.2-10.144.175.254 | n/a           | 00:5b:27:80:01:60 | 00:5b:27:81:01:60        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Strusbek   | fddf:0bf7:80::176:1 | ULA  | 10.144.176.1 | 10.144.176.2-10.144.191.254 | n/a           | 00:5b:27:80:01:76 | 00:5b:27:81:01:76        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Sylsbek    | fddf:0bf7:80::192:1 | ULA  | 10.144.192.1 | 10.144.192.2-10.144.207.254 | n/a           | 00:5b:27:80:01:92 | 00:5b:27:81:01:92        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Trave      | fddf:0bf7:80::208:1 | ULA  | 10.144.208.1 | 10.144.208.2-10.144.223.254 | n/a           | 00:5b:27:80:02:08 | 00:5b:27:81:02:08        | Hetzner (fsn) | ul       | VPN                |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+
| Viehbach   | fddf:0bf7:80::224:1 | ULA  | 10.144.224.1 | 10.144.224.2-10.144.239.254 | n/a           | 00:5b:27:80:02:24 | 00:5b:27:81:02:24        |               |          |                    |
+------------+---------------------+------+--------------+-----------------------------+---------------+-------------------+--------------------------+---------------+----------+--------------------+

Karte
-----

Die Karte kann unter https://map.freifunk-suedholstein.de erreicht werden. Die Karte wird auf dem Gateway Hopfenbach betrieben. Sie basiert auf dem `meshviewer <https://meshviewer.org/>`__ von Freifunk Regensburg. Unsere Konfiguration findet man in unserem fork auf `GitHub <https://github.com/ffsh/meshviewer>`__.

Unsere Grafana instanz ist unter https://map.freifunk-suedholstein.de/grafana erreichbar.


::


    # meshviewer.json
    https://map.freifunk-suedholstein.de/data/meshviewer.json
    # nodes.json (v2)
    https://map.freifunk-suedholstein.de/data/nodes.json
    # nodelist.json
    https://map.freifunk-suedholstein.de/data/nodelist.json


GitHub
------

Fast alle Daten, welche für den Betrieb notwendig sind, werden unter https://github.com/ffsh gespeichert.
