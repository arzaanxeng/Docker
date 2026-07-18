# Docker for AI/ML Engineers 

A phase-wise guide to learning Docker, written for someone coming from an ML/EE background who wants to actually **ship** models and APIs (like a FastAPI recommender system) instead of just running `docker run hello-world` and forgetting everything a week later.

Each phase builds on the last. Do them in order. Don't skip Phase 2 — everything after it assumes you can read a Dockerfile without flinching.

## Phases

| Phase | File | What you'll be able to do after |
|---|---|---|
| 1 | [`01-phase1-basics.md`](01-phase1-basics.md) | Explain what Docker actually is, install it, run/stop/inspect containers |
| 2 | [`02-phase2-dockerfile.md`](02-phase2-dockerfile.md) | Write your own Dockerfile, understand layers & caching, build an image |
| 3 | [`03-phase3-python-ml-images.md`](03-phase3-python-ml-images.md) | Dockerize a real Python/ML app (FastAPI + model file) properly |
| 4 | [`04-phase4-compose-multicontainer.md`](04-phase4-compose-multicontainer.md) | Run multi-service apps (API + DB + cache) with Docker Compose |
| 5 | [`05-phase5-ml-specific-docker.md`](05-phase5-ml-specific-docker.md) | Handle large model files, GPUs, multi-stage builds, secrets, image size |
| 6 | [`06-phase6-deployment-cicd.md`](06-phase6-deployment-cicd.md) | Push images to a registry, deploy to Railway/Render, basic CI/CD |
| 7 | [`07-phase7-advanced-topics.md`](07-phase7-advanced-topics.md) | Debug containers, understand networking/volumes deeply, security basics |

## How to use this repo

- Read a phase, then actually type the commands — don't just skim.
- Each phase ends with a **"You should now be able to..."** checklist and a small hands-on exercise.
- Phase 3 onward uses examples close to a typical ML portfolio project (FastAPI + TF-IDF/sklearn model, similar to a CineMatch-style recommender), so the examples transfer directly to real projects.

## Prerequisites

- Basic command line comfort (cd, ls, cat)
- Docker Desktop (Windows/Mac) or Docker Engine (Linux) installed — covered in Phase 1
- A Python project you don't mind experimenting on (or use the toy example provided in Phase 3)

Once you're through Phase 5, you know enough Docker to comfortably deploy most ML/AI side projects and internship-grade work. Phases 6–7 are the "make it production-decent / interview-decent" layer.
