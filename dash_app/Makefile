PY?=venv/bin/python
PIP?=venv/bin/pip

.PHONY: setup run import-check syntax-check

setup:
	python3.11 -m venv venv
	$(PIP) install --upgrade pip
	$(PIP) install -r requirements.txt

run:
	$(PY) app_dash.py

import-check:
	$(PY) -c "import app_dash; print('import ok; EX=', type(app_dash.EX).__name__)"

syntax-check:
	$(PY) -m py_compile app_dash.py
	@echo 'py_compile ok'

