# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project overview
- This is an Astronomer (Astro) Airflow project, containerized via Docker. The base image is astrocrpublic.azurecr.io/runtime:3.1-1.
- Python version is >=3.12 (pyproject.toml).
- Key components:
  - Airflow DAGs live in dags/ (example: dags/sales_forecast_training.py using the @dag/@task API).
  - Tests live in tests/dags/. There are two styles present:
    - tests/dags/test_dag_example.py (pytest-based checks for import errors, tags, and retries)
    - .astro/test_dag_integrity_default.py (Astro’s DAG integrity harness used by astro dev parse)
  - .astro/config.yaml holds the Astro project name; airflow_settings.yaml is for local-only connections/variables/pools.
  - docker-compose.override.yml extends the default Astro stack with:
    - MLflow tracking server (exposed at port 5001)
    - PostgreSQL backing store for MLflow
    - MinIO S3-compatible object storage (ports 9000/9001) and a helper job to create the mlflow-artifacts bucket
  - Dockerfile installs system build tools often required by ML libraries and sets local-dev env vars for MLflow/MinIO.
- Dependencies appear in both pyproject.toml ([project].dependencies) and requirements.txt. The container build uses requirements.txt; keep these in sync if you author dependencies in pyproject.toml.
- A simple main.py exists for a local hello-world run; Airflow is the primary runtime for orchestrated work.

Common commands (pwsh)
- Start/stop the dev environment (Airflow + services):
  - astro dev start
  - astro dev stop
  - Rebuild the image after changing requirements.txt or Dockerfile: astro dev restart --build
  - Check container status: astro dev ps
  - Tail logs for a service (e.g., scheduler): astro dev logs scheduler
- Airflow CLI inside the dev environment:
  - List DAGs: astro dev run airflow dags list
  - Parse DAGs (uses .astro/test_dag_integrity_default.py): astro dev parse
- Run tests inside the Airflow container (recommended for DAG tests):
  - All tests: astro dev pytest -q
  - Single test by nodeid: astro dev pytest tests/dags/test_dag_example.py::test_dag_retries -q
  - Filtered by pattern: astro dev pytest tests/dags -k dag_retries -q
- Optional: run tests locally (outside containers)
  - If you use uv (uv.lock is present):
    - Install: uv sync
    - Run tests: uv run pytest -q
  - If you prefer venv + pip (ensure system deps for ML libs are present):
    - python -m venv .venv; . .venv/Scripts/Activate.ps1; pip install -r requirements.txt; pytest -q

Architecture and development notes
- DAG structure: dags/sales_forecast_training.py defines a weekly-scheduled DAG using the TaskFlow API. It appends /usr/local/airflow/include to sys.path, so shared Python code can be placed in an include/ directory and imported from tasks.
- Testing expectations: tests/dags/test_dag_example.py requires that DAGs have tags and default_args.retries >= 2. The current sales_forecast_training.py uses retries=1, so this test will fail until default_args["retries"] is increased or the test is adjusted.
- Local services and endpoints (from docker-compose.override.yml):
  - MLflow server: http://localhost:5001
  - MinIO: S3 API at http://localhost:9000; Console at http://localhost:9001
  - The compose override wires these into the same Docker network as Airflow with health checks and a job to create the mlflow-artifacts bucket.
- airflow_settings.yaml is only for local development. Use it to define Connections/Variables/Pools without committing secrets.

What this file does not cover
- No repo-specific lint/format/type-check tooling is configured (e.g., ruff/black/mypy). If you add them later (via pyproject.toml/tool.*), also add the common commands here.
- There are no CI workflows in .github/workflows at this time.
