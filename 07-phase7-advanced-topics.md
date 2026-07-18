# Phase 7 — Advanced-ish / "Decent Level" Topics

This is the last phase. Nothing here is required to ship a working project, but knowing it is what separates "I followed a tutorial" from "I understand what's actually happening" — the kind of thing that comes up in interview follow-up questions.

## Docker networking modes, briefly

By default, Docker creates a **bridge network** for your containers — an internal virtual network where containers get their own internal IP and can talk to each other if explicitly connected (or automatically, if in the same Compose project, as covered in Phase 4).

```bash
docker network ls        # list all networks
docker network inspect bridge
```

Other modes you should at least recognize by name:
- **host** — the container shares the host's network stack directly (no isolation, no port mapping needed, but you lose the isolation benefit). Rarely needed for typical ML APIs.
- **none** — no networking at all, fully isolated. Useful for pure batch/compute jobs that need zero network access.
- **overlay** — used in multi-host container orchestration (Docker Swarm/Kubernetes), lets containers on *different physical machines* talk to each other as if on one network. You won't need this for single-machine deployments, but it's the concept that makes clustering possible.

Creating a custom network manually (useful outside of Compose, e.g. connecting two independently-run containers):

```bash
docker network create my-net
docker run --network my-net --name api ...
docker run --network my-net --name db ...
# 'api' can now reach 'db' by hostname 'db'
```

## Volumes deep-dive

Three types of persistent storage in Docker, in increasing order of "Docker manages it for you":

1. **Bind mount** — maps a specific host path into the container (`-v /home/me/code:/app`). Docker doesn't manage this; it's literally your host filesystem showing up inside the container.
2. **Named volume** — Docker creates and manages storage in its own area (`docker volume create my-vol`, then `-v my-vol:/data`). Portable across containers, not tied to a specific host path.
3. **tmpfs mount** — stored in host memory only, never written to disk, wiped when the container stops. Useful for sensitive temporary data you don't want persisted anywhere.

```bash
docker volume ls              # list all named volumes
docker volume inspect my-vol  # see where it actually lives on disk
docker volume prune           # remove all unused volumes (careful, this is destructive)
```

## Debugging a container that "just doesn't work"

A practical checklist, roughly in the order to check things:

1. `docker ps -a` — did the container even start, or exit immediately?
2. `docker logs <container>` — what did it print before dying? (Most crashes show a clear Python traceback here.)
3. `docker inspect <container>` — check `Env`, `Mounts`, `NetworkSettings` — is an environment variable missing, a mount empty, a port not actually exposed?
4. `docker exec -it <container> bash` (only works if the container is still running) — poke around: is the file actually where you think it is? Is the env var actually set (`echo $VAR`)?
5. If the container exits before you can `exec` into it, override the `CMD` temporarily to keep it alive for inspection:
   ```bash
   docker run -it --entrypoint bash my-image
   ```
   This drops you into a shell instead of running the normal startup command, so you can manually run the failing command step by step and see exactly where it breaks.

## Security basics worth knowing (even if you don't deep-dive)

- **Run as non-root** inside the container (covered in Phase 5) — limits blast radius if compromised.
- **Don't use `:latest` in production** — pin specific versions/tags so a base image update doesn't silently break your build (Phase 6 also touched on this for your own images — applies just as much to base images like `python:3.11-slim` vs `python:latest`).
- **Scan images for known vulnerabilities**:
  ```bash
  docker scout cves my-image:1.0
  ```
  (Docker Desktop includes `docker scout` — it checks your image's layers against known CVE databases. Not something you need to run constantly, but good to know it exists, especially for anything handling real user data.)
- **Minimize what's in the image** — every extra package is extra attack surface, not just extra size (this is one more reason multi-stage builds and slim base images matter beyond just "smaller download").
- **Never trust `COPY . .` blindly** — always double check `.dockerignore` excludes `.git`, `.env`, credentials, and any local secrets before you build an image you intend to push publicly.

## `docker compose` vs Kubernetes — knowing where the ceiling is

Docker Compose is genuinely great for single-machine, single-node setups (dev environments, small deployments, most portfolio-scale projects). It has real limits:

- No automatic scaling across multiple physical machines
- No automatic restart/failover orchestration across a cluster
- No built-in rolling updates / zero-downtime deploys at scale

**Kubernetes (K8s)** is the tool that picks up where Compose's ceiling is — it manages containers across a *cluster* of machines, handles auto-scaling, self-healing (restarting failed containers automatically), rolling deployments, and much more. It's a large enough topic to be its own multi-week study track — not something to tackle in a "phase" here — but it's worth knowing conceptually: **Compose orchestrates containers on one machine; Kubernetes orchestrates containers across a fleet.** If a job posting mentions Kubernetes, this is the honest one-liner to know rather than pretend expertise you don't have yet.

## Interview-ready one-liners (for when someone asks "do you know Docker?")

- "A container is an isolated process sharing the host kernel, not a full virtualized OS like a VM — that's why it's lightweight and starts in seconds."
- "I structure Dockerfiles so dependency installation happens before copying source code, so layer caching means code changes don't force a full dependency reinstall."
- "For ML services I default to `-slim` base images and avoid Alpine because of glibc/musl compatibility issues with compiled scientific packages."
- "I keep secrets out of the image entirely and pass them as environment variables at runtime, since anything baked into a layer via `ENV`/`ARG` is recoverable even if 'overwritten' later."
- "For anything beyond a single machine — scaling, failover, rolling deploys — that's Kubernetes territory; Compose is single-node."

## You should now be able to...

- Explain Docker's networking modes at a conceptual level
- Distinguish bind mounts, named volumes, and tmpfs, and when to use each
- Systematically debug a container that fails to start or behaves unexpectedly
- Name concrete security basics (non-root, pinned tags, image scanning, minimal images)
- Explain, in one sentence, where Compose's ceiling is and what Kubernetes adds

## Where to go from here (optional, beyond "decent")

- Kubernetes fundamentals (pods, deployments, services, ingress) — only if you're targeting infra-heavy ML platform/MLOps roles
- Docker Buildx / cross-platform builds (building an `arm64` image on an `x86` machine, relevant if deploying to Apple Silicon or ARM cloud instances)
- Model-serving-specific tooling that runs *inside* containers (e.g. `bentoml`, `torchserve`, `nvidia-triton`) — these solve serving concerns, not Docker concerns, but often ship as prebuilt Docker images you customize

That's genuinely a solid, practical, interview-defensible Docker foundation for an AI/ML engineer at this point — going further starts trading off against time better spent on ML itself unless you're specifically aiming at MLOps/infra roles.
