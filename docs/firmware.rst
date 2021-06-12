Firmware
========

Die Firmware von Freifunk Südholstein basiert auf `Gluon <https://gluon.readthedocs.io/en/latest/>`_, wir verwenden dabei
in der Regel keine Community-Packages, dadurch ist unsere Firmware sehr stabil und wir können stets auf dem neusten Stand bleiben.

Das site Repository findet man auf `GitHub <https://github.com/ffsh/site>`_.
Freifunk Südholstein hat drei update Kanäle, die regelmäßig bedient werden.

- testing
- rc
- stable

Eine neue Firmware wird stets zuerst im testing Kanal verbreitet, wir haben nur wenige Knoten, welche den testing oder rc Kanal benutzen.
Wir hoffen jedoch, dadurch zumindest grobe Probleme rechtzeitig zu entdecken.

Firmware Releases
*****************

Ein neuer Release beginnt in der Regel mit einem neuen Release von Gluon, dabei können Änderungen an der site Konfiguration nötig werden.
Gelegentlich gibt es auch neue Features, ob diese per Update ausgerollt werden sollen wird dabei mit einer Folgen-Nutzen abschätzung Abgewegt.
Sind die Risiken für Fehlfunktionen zu groß, so wird die Funktion zunächst nicht eingeführt.

Sobald die site config angepasst wurde, wird ein neuer git tag angelegt, die Version setzt sich dabei aus der Gluon-Version sowie einer extra Stelle für
Änderungen von FFSH zusammen.

- Gluon Release = 2020.2.3
- FFSH Release = 0
- FFSH+Gluon Release 2020.2.3.0

Durch die extra Stelle kann FFSH beliebig viele Releases auf Basis derselben Gluon-Version erstellen.
Nachdem ein Release erfolgreich gebaut wurde, wird der Release signiert und per Webserver an die Knoten verteilt.

Dabei gilt die Reihenfolge testing, rc, stable. Nachdem die Knoten eines Kanals aktuallisiert wurden wird in der Regel etwa eine Woche gewartet bis die
Firmware auf den folgenden Kanal ausgerollt wird.
Aufgrund der geringen Anzahl an Knoten werden testing und rc gelegentlich parallel ausgerollt.

Firmware Pipeline
*****************
Für die Firmware Pipeline benutzt Freifunk Südholstein GitHub Actions, sowie abgewandelte Skripte vom Gluon Projekt.

Im `site <https://github.com/ffsh/site>`_ Repository gibt es ein actions Verzeichnis, dort liegen ein paar scripte.

- :code:`generate-actions.py` ist ein leicht abgewandeltes script vom Gluon Projekt, welches den GitHub Workflow generiert.
- :code:`install-dependencies.sh` installiert zusätzliche Pakete in dem Cointainer von GitHub.
- :code:`run-build-local.sh` kann zum bauen auf der lokalen maschine verwendet werden, je nach Bedaf anpassen, für die Fehlersuche gedacht.
- :code:`run-build.sh` ist das eigentliche build Skript, bei einem neuen release wird hier die Version angepasst.

Die vom :code:`generate-actions.py` generierte :code:`build-gluon.yml` Datei liegt in :code:`.github/workflows`, das Skript muss in der Regel nur dann neu ausgeführt werden,
wenn ein neuer Gluon Release ein *target* hinzufügt oder entfernt.
Das Script muss angepasst werden, wenn es einen neuen major Release gibt z.B. 2020->2021, dann muss der tags liste ein neuer regex Ausdruck hinzugefügt werden.

Experimentell ist bisher der Workflow im :code:`release.yml`, das Ziel ist es nach dem erfolgreichen Bauen der Firmware manuell einen Release auf GitHub anzulegen.
Der Workflow soll dann über API ein Firmware Release-Archiv zusammenstellen und als Anhang an den Release beifügen.

Das Bauen der Firmware dauert aktuell etwas mehr als eine Stunde, wenn GitHub durchschnittlich ausgelastet ist. Sollte GitHub sich eines Tages dazu entscheiden, die
Ressourcen zu verringern gibt es die Möglichkeit einen `self-hosted runner <https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners>`_ z.b.
in der Hetzner Cloud zu installieren.

Lokal Bauen
-----------

Die Firmware kann auch lokal gebaut werden, dies ist vor allem nützlich, wenn man einen speziellen release erstellen möchte, oder zur Fehlersuche.

Dafür können natürlich ganz klassisch alle Abhängigkeiten installiert werden, Infos gibt es in der `Gluon Dokumentation <https://gluon.readthedocs.io/en/latest/user/getting_started.html#getting-started>`_.
Allerdings kann es dabei leicht zu Problemen kommen, alternativ liegt im site Repository ein Dockerfile.

.. code-block:: bash

   docker build . --tag gluon
   docker run --name gluon --mount type=bind,source=$(pwd),target=/gluon gluon


Site Repository
***************

Das site Repository wird auf `GitHub <https://github.com/ffsh/site>`__ gehostet. Es gibt einen main branch, auf diesem Branch findet die hauptsächliche Entwicklung statt.
Daneben kann es feature branches geben, die ausgehend vom main branch eine experimentelle oder abweichende Funktionalität mitbringen.

Es gibt kein Changelog im Repository, auf `freifunk-suedholstein.de <https://freifunk-suedholstein.de/tag/firmware/>`_ sind alle Beiträge zur Firmware getagt und man kann einen schnellen Überblick über die Veränderungen bekommen.