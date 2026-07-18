# Phase 6 — Getting Your Image Deployed

Building a good image locally is half the job. This phase covers getting it into the cloud where recruiters/users can actually hit the endpoint.

## Container registries

A **registry** is where images live so other machines (cloud servers) can pull them — like GitHub but for images. Options:

| Registry | Notes |
|---|---|
| **Docker Hub** | Default, free tier available, public or private repos |
| **GitHub Container Registry (GHCR)** | Ties naturally into GitHub repos/Actions |
| **Platform-managed** (Railway, Render, Fly.io) | Many platforms build your image *for you* from your Dockerfile/repo — you often don't need to manually push to a registry at all |

### Pushing to Docker Hub manually

```bash
docker login
docker tag my-app:1.0 yourusername/my-app:1.0
docker push yourusername/my-app:1.0
```

Anyone (or any server) can now pull it:

```bash
docker pull yourusername/my-app:1.0
```

## Deploying to a platform (Railway/Render style — the common path for portfolio projects)

Most modern platforms (Railway, Render, Fly.io) support **"just point me at your Dockerfile"** deployment: you connect your GitHub repo, the platform detects the Dockerfile, builds the image on their infrastructure, and runs it — you never manually `docker push` at all. Key things these platforms need from you:

1. **A working Dockerfile** that binds to `0.0.0.0` and reads the port from an environment variable if the platform requires it:
   ```python
   import os
   port = int(os.environ.get("PORT", 8000))
   ```
   Many platforms inject their own `PORT` env var and expect your app to listen on it — hardcoding `8000` can break deployment if the platform assigns a different port.
2. **Environment variables set in the platform's dashboard** (API keys, DB URLs) — not baked into the image (Phase 5).
3. **A generated public domain** — most platforms require you to explicitly generate/enable a public domain for your service; it's not always automatic.
4. **Correct health check path**, if the platform polls one, so it doesn't think your service is down mid-deploy.

Common first-deploy bugs (worth knowing before you hit them):
- Forgetting an environment variable that exists in your local `.env` but was never added to the platform dashboard → app crashes on boot with a `KeyError`.
- Binding to `127.0.0.1` instead of `0.0.0.0` → platform reports the service as unreachable even though logs show it "started fine".
- Free-tier memory limits (~512MB is common) getting exceeded by a loaded model → container gets OOM-killed, often with a vague "app crashed" message rather than an explicit memory error, so check `docker stats`-equivalent metrics in the platform's dashboard when debugging.

## Basic CI/CD: build & push automatically on every push (GitHub Actions)

Instead of manually building and pushing every time you change code, automate it:

`.github/workflows/docker-build.yml`:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: yourusername/my-app:latest
```

Store `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` as **GitHub repo secrets** (Settings → Secrets and variables → Actions) — never hardcode them in the YAML.

If you're using a platform like Railway that auto-deploys from a connected GitHub repo, you often don't need this workflow at all — pushing to `main` is enough, and the platform's own pipeline handles the build. This GitHub Actions approach becomes more relevant once you're pushing to a registry that isn't tied to an auto-deploying platform, or you want tests to run before an image is even built.

## A minimal "test before build" CI step (good habit)

```yaml
      - name: Run tests
        run: |
          pip install -r requirements.txt
          pytest
```

Add this as a job (or step) before the build/push step, so a broken app never even gets built into an image, let alone deployed.

## Versioning your images sensibly

Avoid only ever using `:latest` — it makes rollbacks hard because you can't tell what `:latest` actually was yesterday. Tag with something meaningful:

```bash
docker build -t yourusername/my-app:1.2.0 .
docker build -t yourusername/my-app:$(git rev-parse --short HEAD) .
```

Using the short git commit hash as a tag means you can always trace exactly which code produced a given image.

## You should now be able to...

- Push an image to Docker Hub (or explain why you didn't need to, for platform-managed deploys)
- Explain the common first-deploy bugs (binding, ports, env vars, memory) before hitting them blind
- Set up a basic GitHub Actions workflow to build and push an image automatically
- Explain why `:latest` alone is a poor versioning strategy

## Exercise

Push your Phase 5 image to Docker Hub manually first. Then write a GitHub Actions workflow that does it automatically on every push to `main`, using repo secrets for credentials. Confirm it runs by checking the Actions tab after pushing a commit.

Next: [Phase 7 — Advanced/decent-level topics: networking, volumes deep-dive, and security](07-phase7-advanced-topics.md)
