.. toctree::
   :maxdepth: 2
   :caption: Inhalte

ELK Server
==========

ELK ist ein Softwarestack aus Eleasticsearch, Logdash und Kibana. Es eignet sich um logs von verschiedenen Servern oder auch docker Containern zentral zu sammeln und zu visualisieren.

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
Für logstash müssen wir ein paar Konfigurationen mehr anlegen.

::

   /etc/logstash/conf.d
   
Als erstes definieren wir den Input der Kette.

::

    02-filebeat-input.conf

::
   
   input {
       beats {
           port => 5044
           ssl => true
           ssl_certificate => "/etc/letsencrypt/live/mon.freifunk-suedholstein.de/fullchain.pem"
           ssl_key => "/etc/letsencrypt/live/mon.freifunk-suedholstein.de/privkey.pem"
       }
    }

::

   10-syslog-filter.conf
   
:: 

   filter {
      if [type] == "syslog" {
         grok {
            match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
            add_field => [ "received_at", "%{@timestamp}" ]
            add_field => [ "received_from", "%{host}" ]
         }
         syslog_pri { }    
         date {
            match => [ "syslog_timestamp", "MMM d HH:mm:ss", "MMM dd HH:mm:ss" ]
         }
      }
   }

::

   30-elasticsearch-output.conf
   
::
   
   output {
      elasticsearch {
          hosts => ["localhost:9200"]
          sniffing => true
          manage_template => false
          index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
          document_type => "%{[@metadata][type]}"
      }   
   }

kibana
------
Auch hier müssen wir nicht viel machen, wir bearbeiten nur einen Wert in der Konfiguration.

::

   /etc/kibana/kibana.yml


:: 

   server.host: "localhost"


nginx
-----

.. literalinclude:: configs/nginx-kibana.conf

