# Hello FastAPI

This is a minimal FastAPI app.

Files:
- `main.py` — FastAPI application
- `requirements.txt` — Python dependencies
- `test_main.py` — simple pytest test using FastAPI TestClient

Quick start (macOS / zsh):

1. Create and activate a venv:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Run the app:

```bash
uvicorn main:app --reload
```

Open `http://127.0.0.1:8000` and interactive docs at `http://127.0.0.1:8000/docs`.

4. Run tests:

```bash
pytest -q
```

## Docker

Build the Docker image locally:

```bash
# from project root
docker build -t hello-fastapi .
```

Run the image (maps port 8000):

```bash
docker run --rm -p 8000:8000 hello-fastapi
```

Or use Docker Compose:

```bash
docker-compose up --build
```

Then open `http://127.0.0.1:8000` and `http://127.0.0.1:8000/docs` as usual.

## Kubernetes

Kubernetes manifests live in the `k8s/` folder. See `k8s/README.md` for Minikube and registry instructions, access methods (minikube service, port-forward, NodePort), and troubleshooting tips.

Link: `k8s/README.md`
