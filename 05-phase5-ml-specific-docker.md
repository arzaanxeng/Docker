# Phase 5 — ML-Specific Docker: GPUs, Multi-Stage Builds, and Image Size

This is the phase that separates "I can Dockerize a toy app" from "I can Dockerize something an ML team would actually deploy."

## Multi-stage builds — shrinking your image dramatically

Some Python packages (torch, some C-extension-heavy libraries) need build tools (compilers, headers) to install, but those build tools are dead weight at runtime. A **multi-stage build** lets you use a heavy "builder" stage to compile/install things, then copy only the finished artifacts into a slim final image — throwing away the build tools entirely.

```dockerfile
# ---- Stage 1: builder ----
FROM python:3.11 AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ---- Stage 2: final, slim runtime image ----
FROM python:3.11-slim

WORKDIR /app

# Copy only the installed packages from the builder stage
COPY --from=builder /root/.local /root/.local
COPY . .

ENV PATH=/root/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

The final image never contains the full Debian build toolchain from the `builder` stage — only the slim base plus the installed Python packages. This can cut image size significantly, especially for packages with heavy build dependencies.

## GPU access inside containers

By default, a container has **no access to your GPU**, even if you have one. To use a GPU inside Docker, you need:

1. An NVIDIA GPU + NVIDIA drivers installed on the **host** machine
2. The **NVIDIA Container Toolkit** installed on the host (this is what bridges the driver into containers)
3. A CUDA-enabled base image inside the Dockerfile, e.g.:

```dockerfile
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04
```

4. Running the container with GPU access explicitly requested:

```bash
docker run --gpus all my-gpu-image
```

Or in Compose:

```yaml
services:
  train:
    build: .
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

**Important distinction**: the *driver* lives on the host and must match what your host's actual GPU supports — it is never installed inside the container. The container only needs the CUDA *toolkit/runtime* libraries matching a driver version the host already supports. Mismatches here (host driver too old for the CUDA version in the image) are the #1 cause of "it says CUDA not available" issues.

For most portfolio-level / free-tier cloud deployments (Railway, Render free tiers, etc.), you will **not** have GPU access — this section is more relevant once you're training on your own machine with an NVIDIA GPU, or using a GPU-enabled cloud instance.

## Reducing image size — a checklist

1. Use `-slim` (or a purpose-built minimal image) instead of the full base image.
2. Use multi-stage builds for anything with compiled dependencies.
3. Always `--no-cache-dir` with `pip install`.
4. Combine `RUN` commands with `&&` where it makes sense, to avoid extra layers (minor effect, but good practice):
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends \
       libgomp1 \
       && rm -rf /var/lib/apt/lists/*
   ```
   Note `rm -rf /var/lib/apt/lists/*` in the **same** `RUN` instruction — if you delete it in a separate `RUN`, the earlier layer still contains the full apt cache, so the space isn't actually reclaimed.
5. Don't install `dev` extras / testing dependencies (pytest, jupyter, etc.) in your production image. Keep a separate `requirements-dev.txt`.
6. Check what you actually shipped:
   ```bash
   docker images                     # see the final size
   docker history my-image:1.0       # see size contributed by EACH layer
   ```
   `docker history` is genuinely useful — it'll show you exactly which instruction bloated your image (often it's an unnecessary `apt-get install` or a forgotten large file).

## Handling secrets properly at build time

Never do this:

```dockerfile
# BAD — secret gets permanently baked into an image layer, extractable forever
ENV TMDB_API_KEY=abc123
```

Even if you remove it in a later instruction, it's still recoverable from the intermediate layer. Options that actually work:

- Pass secrets at **runtime** only (`docker run -e` or `--env-file`, or your cloud platform's "environment variables" settings) — never bake them into the image at all. This is the right approach for almost all ML API deployments.
- For build-time secrets you genuinely need during the build (rare for most ML apps), use Docker's `--secret` mount feature with BuildKit, which never persists the secret into a layer. This is more advanced and usually unnecessary unless you're, say, authenticating to a private package registry during `pip install`.

## `HEALTHCHECK` — telling Docker (and your deployment platform) if your app is actually alive

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/health || exit 1
```

This requires an actual `/health` endpoint in your app that returns 200 OK. Many deployment platforms (Railway, Render, Kubernetes) poll a health check endpoint to decide whether to route traffic to your container, restart it, or mark it unhealthy — so this isn't just cosmetic, it directly affects uptime behavior in production.

## Non-root user (a security basic worth knowing even at "decent" level)

By default, containers run as root. If someone breaks out of your app through a vulnerability, running as root inside the container gives them more to work with. Good practice:

```dockerfile
FROM python:3.11-slim

RUN useradd --create-home appuser
WORKDIR /home/appuser/app

COPY --chown=appuser:appuser . .
RUN pip install --no-cache-dir -r requirements.txt

USER appuser

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## You should now be able to...

- Write a multi-stage Dockerfile to shrink image size
- Explain what's needed (host-side and image-side) for GPU access in a container
- Use `docker history` to find what's bloating an image
- Explain why secrets should never be baked into image layers via `ENV`/`ARG`
- Add a basic `HEALTHCHECK` and understand why deployment platforms care about it
- Run a container as a non-root user

## Exercise

Take your Phase 3 image, check its size with `docker images`. Rewrite the Dockerfile as a multi-stage build, rebuild, and compare the size difference. Then add a `/health` endpoint to your FastAPI app and a corresponding `HEALTHCHECK` instruction.

Next: [Phase 6 — Getting your image deployed (registries, Railway/Render, CI/CD basics)](06-phase6-deployment-cicd.md)
