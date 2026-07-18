# Phase 2 — Writing Your Own Dockerfile

A **Dockerfile** is a recipe: a plain text file with step-by-step instructions for building an image. `docker build` reads it top to bottom and produces an image.

## A minimal Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["python3", "app.py"]
```

Let's go instruction by instruction.

### `FROM`

Every Dockerfile starts with a base image — you're never building from absolute scratch (well, you technically can with `FROM scratch`, but that's rare and advanced). `python:3.11-slim` means: start from an official Python 3.11 image, `slim` variant (a smaller version with fewer preinstalled OS packages).

### `WORKDIR`

Sets the working directory inside the container for all following instructions. If it doesn't exist, Docker creates it. Everything after this (`COPY`, `RUN`, `CMD`) happens relative to `/app`.

### `COPY`

Copies files from your machine (the "build context") into the image.

```dockerfile
COPY . .
```

means "copy everything in the current directory (where you run `docker build`) into `/app` inside the image". You can also copy specific files:

```dockerfile
COPY requirements.txt .
COPY app.py .
```

### `RUN`

Executes a command **at build time**, and the result gets baked into the image as a new layer. This is how you install dependencies.

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

### `CMD`

The default command that runs **when a container starts** (not at build time). There can only be one effective `CMD` per Dockerfile (the last one wins).

```dockerfile
CMD ["python3", "app.py"]
```

Note the array/"exec" form `["python3", "app.py"]` is preferred over the string form `python3 app.py` — it doesn't invoke a shell, so signals (like Ctrl+C / `docker stop`) reach your process directly.

## Building the image

```bash
docker build -t my-app:1.0 .
```

- `-t my-app:1.0` tags the image with a name and version
- `.` is the build context — the directory Docker sends to the daemon to build from (this is why you should have a `.dockerignore`, covered below)

Run it:

```bash
docker run -p 8000:8000 my-app:1.0
```

## Layers and caching (the part that actually matters for speed)

Every instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, etc.) creates a **layer**. Docker caches layers and reuses them if nothing has changed, which makes rebuilds fast — *if you order your Dockerfile correctly*.

**Bad ordering** (common beginner mistake):

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python3", "app.py"]
```

Here, `COPY . .` copies your *entire* codebase, including files that change every time you edit code (like `app.py`). Since Docker invalidates the cache for a layer once anything above it changes, changing a single line in `app.py` invalidates the cache for `COPY . .`, which then invalidates `RUN pip install...` too — meaning **every single code change triggers a full reinstall of every dependency**. Painfully slow.

**Better ordering:**

```dockerfile
FROM python:3.11-slim
WORKDIR /app

# Copy ONLY the dependency file first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the code AFTER dependencies are installed
COPY . .

CMD ["python3", "app.py"]
```

Now, as long as `requirements.txt` doesn't change, Docker reuses the cached "pip install" layer even if you change `app.py` a hundred times. Only touching `requirements.txt` triggers a reinstall.

**Rule of thumb: put things that change rarely (dependencies) near the top, and things that change often (your source code) near the bottom.**

## `.dockerignore`

Just like `.gitignore`, this file tells Docker which files/folders to exclude from the build context — so you don't accidentally bloat your image or leak secrets.

```
__pycache__/
*.pyc
.git
.venv
.env
*.ipynb_checkpoints
node_modules
```

Without this, `COPY . .` might drag in your entire `.git` history, virtual environment folders, or Jupyter checkpoint junk — bloating build time and image size for no reason.

## `EXPOSE` (documentation, not enforcement)

```dockerfile
EXPOSE 8000
```

This is purely documentation — it tells humans (and some tooling) "this container listens on port 8000". It does **not** actually publish the port; you still need `-p` on `docker run` for that.

## ENV — setting environment variables inside the image

```dockerfile
ENV PYTHONUNBUFFERED=1
ENV MODEL_PATH=/app/models/model.pkl
```

`PYTHONUNBUFFERED=1` is a very common one for Python apps in Docker — it forces stdout/stderr to flush immediately instead of buffering, so your `print()` / logs actually show up in `docker logs` in real time.

## A slightly more complete real Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Note `--host 0.0.0.0` — inside a container, binding to `127.0.0.1`/`localhost` means the server is only reachable from *inside* the container itself. You must bind to `0.0.0.0` so the port mapping (`-p`) can actually reach it from outside.

## Useful inspection commands

```bash
docker logs <container>              # see stdout/stderr of a running/stopped container
docker logs -f <container>           # follow logs live
docker exec -it <container> bash     # get a shell INSIDE a running container
docker inspect <container>           # full JSON metadata (env vars, mounts, network, etc.)
```

`docker exec -it <container> bash` is one of the most useful debugging commands you'll use — it lets you poke around inside a *running* container to check if a file actually got copied, if an env var is set correctly, etc.

## You should now be able to...

- Write a basic Dockerfile for a Python app
- Explain why instruction order affects build speed (layer caching)
- Use `.dockerignore` correctly
- Explain the difference between `RUN` (build time) and `CMD` (container start time)
- Get a shell inside a running container to debug it

## Exercise

Take any small Python script (even a "hello world" Flask/FastAPI app), write a Dockerfile for it following the caching-friendly order, build it, and run it with a port mapping. Then intentionally break the ordering (put `COPY . .` before `pip install`), rebuild after a trivial code change, and time the difference with `time docker build -t test .`.

Next: [Phase 3 — Dockerizing a real Python/ML app](03-phase3-python-ml-images.md)
