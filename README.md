# Hello FastAPI — Kubernetes Demo

A minimal FastAPI service packaged for Docker and Kubernetes. Used as a small, end-to-end
reference for the full path: local dev → container image → cluster deployment.

## Endpoints

| Method | Path | Description | Example response |
|--------|------|-------------|------------------|
| `GET` | `/` | Health / welcome message | `{"message": "Welcome to FastAPI!"}` |
| `GET` | `/items/{item_id}` | Echo an item id, optional `?q=` query string | `{"item_id": 42, "q": "test"}` |
| `GET` | `/docs` | Swagger UI (interactive API docs) | HTML |
| `GET` | `/redoc` | ReDoc API reference | HTML |
| `GET` | `/openapi.json` | Raw OpenAPI schema | JSON |

`item_id` is typed as `int` — a non-numeric value returns `422 Unprocessable Entity`.

## Project structure

```
hello-fastapi-k8s/
├── main.py              # FastAPI application (2 routes)
├── test_main.py         # Smoke test for `/` using FastAPI TestClient
├── requirements.txt     # Python dependencies
├── Dockerfile           # python:3.11-slim, non-root user, port 8000
├── .dockerignore        # Build-context exclusions (.venv, __pycache__, .git)
├── docker-compose.yml   # Single-service compose file
├── k8s/
│   ├── deployment.yaml  # 2 replicas, readiness + liveness probes
│   ├── service.yaml     # ClusterIP service, port 80 → targetPort 8000
│   └── README.md        # Minikube / registry / access instructions
└── README.md
```

## Local development

Requires Python 3.10+. The working virtualenv in this repo (`.venv`) runs Python 3.13.1,
but 3.10 is the real floor: `pytest 9.0.1` and `starlette 0.50.0` both declare
`requires-python >=3.10` (`fastapi` needs >=3.8, `uvicorn` >=3.9, `pydantic` >=3.9).

Create and activate a virtualenv:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

Run the development server with auto-reload:

```bash
uvicorn main:app --reload
```

Then open <http://127.0.0.1:8000> and the interactive docs at <http://127.0.0.1:8000/docs>.

Run the tests:

```bash
pytest -q
```

This passes with 1 test — `test_main.py`, a smoke test for `/` using the FastAPI
`TestClient`.

### Pinning dependencies

`requirements.txt` is unpinned: it lists `fastapi`, `uvicorn[standard]`, `pytest`, and
`httpx` with no version constraints, so a fresh install resolves to whatever is current.
The versions resolved in the working `.venv` are:

| Package | Version |
|---------|---------|
| `fastapi` | 0.123.5 |
| `uvicorn` | 0.38.0 |
| `starlette` | 0.50.0 |
| `pydantic` | 2.12.5 |
| `pytest` | 9.0.1 |
| `httpx` | 0.28.1 |

For reproducibility, pin these (`fastapi==0.123.5`, and so on) or capture the full
resolved set with `pip freeze > requirements.txt`. Note the repo's `.dockerignore`
already excludes `.venv`, `__pycache__`, `*.pyc`, `*.pyo`, `*.pyd`, `env/`, `venv/`,
`.env`, `.git`, and `.gitignore` from the Docker build context, so the local virtualenv
does not leak into images.

## Configuration

The app itself takes no configuration. The container reads one environment variable:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `8000` | Informational; the `CMD` hardcodes `--port 8000` and `EXPOSE 8000`. |

## Docker

```bash
# build
docker build -t hello-fastapi:latest .

# run
docker run --rm -p 8000:8000 hello-fastapi:latest

# or with Compose
docker-compose up --build
```

The image runs as a non-root `appuser` and serves with
`uvicorn main:app --host 0.0.0.0 --port 8000`. The build context is trimmed by
`.dockerignore`, which excludes `.venv/`, `__pycache__/`, and `.git`.

## Kubernetes

Manifests live in `k8s/`. Full Minikube, registry, port-forward, and NodePort
instructions are in [`k8s/README.md`](k8s/README.md).

Quick path with Minikube (build the image inside the cluster's daemon so no registry is needed):

```bash
eval "$(minikube -p minikube docker-env)"
docker build -t hello-fastapi:latest .
kubectl apply -f k8s/
kubectl get pods -l app=hello-fastapi
```

The Deployment runs 2 replicas behind a ClusterIP Service (`port 80` → `targetPort 8000`)
with readiness and liveness probes against `/`.

Reach it locally:

```bash
kubectl port-forward svc/hello-fastapi-service 8080:80
# then open http://127.0.0.1:8080/
```

## Notes and gotchas

- **`imagePullPolicy: IfNotPresent`** is set in `deployment.yaml`. This is what lets the cluster
  use a locally built image instead of trying to pull `hello-fastapi:latest` from a registry.
- **The Service is `ClusterIP`**, so it is not reachable from outside the cluster until you
  port-forward it (or patch it to `NodePort`). See `k8s/README.md`.
- **The probes hit `/`**, which returns `200` as soon as the app starts — no external
  dependencies to wait on.
