.. toctree::
   :maxdepth: 2
   :caption: Inhalte

ELK Server
==========

ELK ist ein Softwarestack aus Eleasticsearch, Logdash und Kibana. Es eignet sich um logs von verschiedenen Servern oder auch docker Coatainern zentral zu sammeln und zu visualisieren.

Elaststic-apt Repository anlegen
--------------------------------

Diese Anleitung ist auf Ubuntu 18.04 und Elastic 7.x ausgelegt

Als erstes fügen wir das Elastic Repository hinzu.

::

   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
   echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

Für unser Setup benötigen wir außer den Hauptkomponenten noch eine jre, nginx und die apache2-utils.

::
   apt update
   apt install default-jre-headless elasticsearch kibana logstash nginx apache2-utils


elasticsearch
-------------
Hier müssen wir nicht viel machen, wir bearbeiten nur einen Wert in der Konfiguration.
::
   
   /etc/elasticsearch/elasticsearch.yml

::
   
   network.host: localhost

logstash
--------

kibana
------
Auch hier müssen wir nicht viel machen, wir bearbeiten nur einen Wert in der Konfiguration.

::

   /etc/kibana/kibana.yml
   
:: 

   server.host: "localhost"


nginx
-----




