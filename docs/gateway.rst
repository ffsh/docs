.. toctree::
   :maxdepth: 2
   :caption: Inhalte

Gateway Konfiguration
=====================
Dies ist die Dokumentation für die Gateways von Freifunk Südholstein.

.. image:: images/overview.png

In 2021 haben wir für die Einrichtung der Gateways auf Ansible umgestellt.
Davor war die Einrichtung eines Gateways ein manueller Prozess der bereits bei kleineren Fehlern zu stundenlangen Rätseln führen konnte.

Ansible verkürzt die Einrichtung eines Gateways auf etwa 30min.

Unser Ansible Repository liegt auf GitHub: https://github.com/ffsh/ansible

Ansible installieren
--------------------
Ansible muss auf deinem lokalen system installiert werden.
In den meisten Linux Distributionen wirst du ein oder mehrere Ansible pakete finden.
Unter Manjaor ist das "ansible" paket ausreichend.
Mehr optionen findest du hier:
https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html


Struktur
--------
Unser Ansible Repository ist relativ einfach gehalten, dadurch bleibt es übersichtlich.

setup.yml
^^^^^^^^^
Hier werden neben ein paar Einstellungen vor allem die Rollen definert und in welcher Reihenfolge sie auf dem Gateway angewendet werden sollen.

hosts.yml
^^^^^^^^^
Hier Werden alle variablen, welche von den Rollen benötigt werden gespeichert. Abgesehen von den Geheimen natürlich.

host_vars/
^^^^^^^^^^
Hier gibt es pro Gateway eine Datei mit verschlüsselten Inhalt mit den geheimen Daten.

roles/
^^^^^^
Hier werden die Rollen gespeichert, wenn etwas nicht funktioniert liegt der Fehler vermutlich hier.

Secrets anlegen und bearbeiten
------------------------------
Für unser Gateway brauchen wir natürlich auch Geheime daten, welche wir nicht öffentlich im git Repository ablegen wollen.
Dafür bietet Ansible eine integrierte Funktion, welche eine verschlüsselte Datei mit den gewünschten Daten erstellt.
Die Readme Datei im Repository erklärt wie neue Secrets erstellt werden und wie man sie bearbeitet.

Mehr zu diesem Thema: https://docs.ansible.com/ansible/latest/user_guide/vault.html#


Hinweise zur Nutzung
--------------------