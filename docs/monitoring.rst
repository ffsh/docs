.. toctree::
   :maxdepth: 2
   :caption: Inhalte

monitoring
==========

Diese Dokumentation behandelt die Konfiguration unseres monitoring servers https://stats.freifunk-suedholstein.de

Vorraussetzungen
----------------

Der aktuelle Server läuft mit Debian 12 in der Hetzner Cloud auf einer CX11 instanz (1VCPU 2GB RAM 20GB disk)


Prometheus
----------

Prometheus ist das Herz des monitoring systems, Prometheus sammelt die Daten von den Gateways und anderen Servern und speichert sie in seiner eigenen TSDB.
Prometheus stellt eine API bereit mit der Daten aus der TSDB abgefragt werden können.

Wir verwenden die Prometheus version aus dem Debian repository.

::


   apt install prometheus

Die Konfiguration liegt in :code:`/etc/prometheus/prometheus.yml`

Der systemd service kann per :code:`systemctl status prometheus.service` geprüft werden.

Die Konfiguration:

::

   # my global config
   global:
   scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
   evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
   # scrape_timeout is set to the global default (10s).

   # Alertmanager configuration
   alerting:
   alertmanagers:
      - static_configs:
         - targets:
            # - alertmanager:9093

   # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
   rule_files:
   # - "first_rules.yml"
   # - "second_rules.yml"

   # A scrape configuration containing exactly one endpoint to scrape:
   # Here it's Prometheus itself.
   scrape_configs:
   # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
   - job_name: "prometheus"

      # metrics_path defaults to '/metrics'
      # scheme defaults to 'http'.

      static_configs:
         - targets: ["localhost:9090"]

   - job_name: "gateways"

      metrics_path: stats/metrics
      scheme: https
      basic_auth:
         username: '...'
         password: '...'

      static_configs:
         - targets: ["barnitz.freifunk-suedholstein.de", "beste.freifunk-suedholstein.de", "bille.freifunk-suedholstein.de", "brunsbach.freifunk-suedholstein.de", "heilsau.freifunk-suedholstein.de", "sylsbek.freifunk-suedholstein.de", "trave.freifunk-suedholstein.de"]

   - job_name: "services"

      metrics_path: stats/metrics
      scheme: https

      static_configs:
         - targets: ["mail.freifunk-suedholstein.de", "map.freifunk-suedholstein.de", "web.freifunk-suedholstein.de"]

Username und Passwort sind in unserer KeePass Datei.

Zusätzlich beschränken wir Prometheus darauf auf localhost zu lauschen.

:code:`/etc/default/prometheus`

::

   # Set the command-line arguments to pass to the server.
   # Due to shell escaping, to pass backslashes for regexes, you need to double
   # them (\\d for \d). If running under systemd, you need to double them again
   # (\\\\d to mean \d), and escape newlines too.
   ARGS="--web.listen-address=127.0.0.1:9090"

Diese Einstellungen werden von systemd bei einem Neustart geladen.

:code:`systemctl restart prometheus.service`

Die Daten von prometheus liegen in:

:code:`/var/lib/prometheus/metrics2`

Grafana
-------

Grafana wird verwendet um die Daten aus prometheus zu visualiseren.
Die installation und Konfiguration ist hier nicht detailiert.

Die wesentlichen punkte sind.

- grafana installation https://grafana.com/docs/grafana/latest/setup-grafana/installation/
- ngninx reverse proxy, letsencrypt
- prometheus als Daten-Quelle konfigurieren
- grafana konfiguration anpassen, grafana sollte nur auf localhost lauschen
- dashboards einrichten, für den node_exporter gibt es fertige dashboards aus der Community