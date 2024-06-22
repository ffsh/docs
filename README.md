[![Documentation Status](https://readthedocs.org/projects/freifunk-suedholstein/badge/?version=latest)](https://freifunk-suedholstein.readthedocs.io/de/latest/?badge=latest)

# docs
In diesem Repository findest du die Dokumentation über Freifunk Südholstein.

Du kannst sie auf [docs.freifunk-suedholstein.de](https://docs.freifunk-suedholstein.de) lesen.

# Lokal bauen

Python virtual environment erstellen.

`python -m venv .venv`

Virtuelle Umgebung aktivieren.

`. .venv/bin/activate`

Pakete installieren.

`pip install sphinx sphinx-rtd-theme --upgrade`

Baue die HTML Version.
`make html`

Das Resultat ist in `_build` und kann mit deinem Browser angesehen werden, öffne die index.html oder eine der anderen HTML Dateien.