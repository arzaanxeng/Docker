# Phase 3 — Dockerizing a Real Python/ML App

This phase uses a realistic example: a **FastAPI service that serves predictions from a trained model** (e.g. a TF-IDF-based recommender, or a scikit-learn classifier saved as a `.pkl`). This mirrors what most ML portfolio projects actually look like once deployed.

## Example project structure

```
my-ml-api/
├── app.py
├── requirements.txt
├── models/
│   └── model.pkl
├── .dockerignore
└── Dockerfile
```

`app.py` (simplified):

```python
from fastapi import FastAPI
import pickle

app = FastAPI()

with open("models/model.pkl", "rb") as f:
    model = pickle.load(f)

@app.get("/predict")
def predict(query: str):
    result = model.recommend(query)
    return {"result": result}
```

`requirements.txt`:

```
fastapi
uvicorn
scikit-learn
pandas
```

## The Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build and run:

```bash
docker build -t ml-api:1.0 .
docker run -p 8000:8000 ml-api:1.0
```

Test it from your host machine:

```bash
curl "http://localhost:8000/predict?query=inception"
```

## Choosing a base image (this actually matters for ML)

| Base image | Size (rough) | When to use |
|---|---|---|
| `python:3.11` | ~900MB+ | Full Debian, has more system tools preinstalled. Use if you hit missing-library errors and don't want to debug which ones. |
| `python:3.11-slim` | ~150MB | Debian-based but stripped down. **Good default for most ML APIs.** |
| `python:3.11-alpine` | ~50MB | Extremely small, but uses `musl` libc instead of `glibc`. **Warning**: many ML packages (numpy, scipy, scikit-learn, torch) either fail to install or behave subtly differently on Alpine because prebuilt wheels are compiled against glibc. Avoid Alpine unless you know what you're doing. |
| `nvidia/cuda:...` | Several GB | Needed when you actually require GPU/CUDA inside the container (covered in Phase 5). |

**Practical takeaway: use `-slim` as your default for ML/AI apps. Avoid Alpine unless every dependency is pure Python.**

## Why `pip install --no-cache-dir`?

Without `--no-cache-dir`, pip keeps a local download cache inside the image, which just adds dead weight — you're never going to `pip install` again inside that running container, so the cache is pure waste. Always use `--no-cache-dir` in Docker builds.

## Handling large model files

If `model.pkl` is small (a few MB), just `COPY` it in like any other file. If it's large (hundreds of MB to a few GB — common for deep learning checkpoints), consider:

- **Don't commit large model files to Git** at all — use Git LFS, or better, download the model at container startup/build time from cloud storage (S3, HF Hub, GDrive) instead of baking it into the image.
- If you do bake it in, put the `COPY models/ ./models/` instruction as late as possible in the Dockerfile (after installing dependencies) so code changes don't force a re-copy of a multi-GB file needlessly — though note Docker's cache does key off file content hash, so an unchanged model file layer is still reused even if it's listed early. The real reason to keep it below `pip install` is dependency-install caching, discussed in Phase 2.

## Memory awareness (important for free-tier deployments)

When you load a pickle file into memory (`pickle.load`), that memory usage is *per running instance* of your container. Free tiers on platforms like Railway/Render often cap RAM around 512MB. A model that's 200MB on disk can easily balloon in RAM once deserialized (sklearn/pandas objects carry overhead). Always check your container's actual memory usage under load, not just the file size on disk:

```bash
docker stats
```

This shows live CPU/memory usage per running container — genuinely useful for catching an OOM (out-of-memory) problem before your host does.

## Environment variables for config (don't hardcode secrets/paths)

Instead of hardcoding API keys or model paths in `app.py`, read them from environment variables:

```python
import os
API_KEY = os.environ["TMDB_API_KEY"]
MODEL_PATH = os.environ.get("MODEL_PATH", "models/model.pkl")
```

Pass them in at `docker run` time:

```bash
docker run -p 8000:8000 -e TMDB_API_KEY=abc123 ml-api:1.0
```

Or from a file (much more common in practice):

```bash
docker run -p 8000:8000 --env-file .env ml-api:1.0
```

`.env` file:

```
TMDB_API_KEY=abc123
MODEL_PATH=models/model.pkl
```

**Never `COPY` a `.env` file with real secrets into an image.** Anyone who pulls that image can extract the secrets from its layers. Keep `.env` in `.dockerignore` and pass secrets at runtime instead.

## Connection pooling gotcha (a real bug class you'll hit)

If your FastAPI app calls an external API (like TMDB) on every request, creating a brand-new HTTP client per request (`httpx.AsyncClient()` inside the route function) exhausts connections under load and causes intermittent 502s. Instead, create **one shared client** at startup and reuse it, using FastAPI's `lifespan` context manager:

```python
from contextlib import asynccontextmanager
import httpx

client: httpx.AsyncClient | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global client
    client = httpx.AsyncClient(timeout=10.0)
    yield
    await client.aclose()

app = FastAPI(lifespan=lifespan)
```

This is a Python/FastAPI concern more than a Docker one, but it's the kind of bug that only shows up once you're actually running under container constraints (limited connections, limited memory) — worth knowing at this phase.

## You should now be able to...

- Dockerize a FastAPI + ML model service end-to-end
- Pick a sensible base image and explain why Alpine is often a trap for ML workloads
- Pass secrets/config via environment variables instead of hardcoding them
- Use `docker stats` to sanity-check memory usage
- Explain why baking large model files into images has tradeoffs

## Exercise

Take one of your own trained models (or train a tiny toy one — e.g. `sklearn`'s iris classifier), wrap it in a minimal FastAPI endpoint, Dockerize it following this phase, and hit it with `curl` from your host machine. Then run `docker stats` while sending it a few requests and note the memory usage.

Next: [Phase 4 — Docker Compose for multi-container apps](04-phase4-compose-multicontainer.md)
