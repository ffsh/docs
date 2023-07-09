[![Documentation Status](https://readthedocs.org/projects/freifunk-suedholstein/badge/?version=latest)](https://freifunk-suedholstein.readthedocs.io/de/latest/?badge=latest)

# docs
In diesem Repository findest du die Dokumentation über Freifunk Südholstein.

Du kannst sie auf [docs.freifunk-suedholstein.de](https://docs.freifunk-suedholstein.de) lesen.

# Lokal bauen

Python virtual environment erstellen.

`python -m venv .venv`

Virtuelle umgebung aktivieren.

`. .venv/bin/activate`

Pakete installieren.

`pip install sphinx sphinx-rtd-theme`

Wechsel in die docs directory

`cd docs`

Baue die html version.
`make html`

Das resultat ist in `docs/_build` und kann mit deinem Browser angesehen werden, öffne die index.html oder eine der anderen html dateien.