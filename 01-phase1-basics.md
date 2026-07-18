# Phase 1 — What Docker Actually Is (and Your First Containers)

## The problem Docker solves

You've probably lived this: your ML script works fine on your laptop, but breaks on a friend's machine or on a cloud server, because of:

- A different Python version
- Missing system libraries (`libgomp`, `ffmpeg`, CUDA drivers, etc.)
- A different OS
- "Works on my machine" — the most cursed sentence in software

Docker solves this by packaging your app **and everything it needs to run** (Python version, system libraries, dependencies, your code) into one portable unit called an **image**. That image runs identically on your laptop, your friend's laptop, or a server in the cloud — because it's not relying on whatever happens to be installed on the host machine.

## Container vs Virtual Machine (the analogy that actually matters)

A **Virtual Machine (VM)** virtualizes an entire computer — it runs its own full OS kernel on top of a hypervisor. Heavy, slow to boot (minutes), each VM might be gigabytes.

A **container** does not run its own OS kernel. It shares the host machine's kernel and just isolates the process (its filesystem, network, processes) using Linux features (namespaces & cgroups). This makes containers:

- Lightweight (megabytes, not gigabytes)
- Fast to start (seconds, sometimes less than a second)
- Still isolated enough that "it works in the container" ≈ "it works everywhere"

Think of a VM as building a whole separate house. A container is more like a apartment in a building — separate living space, shares the building's foundation/plumbing (the kernel).

## Core vocabulary (learn these cold)

| Term | Meaning |
|---|---|
| **Image** | A read-only blueprint/template — your app + its dependencies + OS libraries, frozen as layers. Doesn't run by itself. |
| **Container** | A running (or stopped) *instance* of an image. Like a class vs an object — image is the class, container is the object. |
| **Dockerfile** | A text file with instructions to build an image (covered in Phase 2). |
| **Docker Hub / Registry** | A place to store and pull images, like GitHub but for images (e.g. `python:3.11-slim` is pulled from Docker Hub). |
| **Volume** | A way to persist data outside a container's lifecycle (so data survives container deletion). |
| **Docker Engine / Daemon** | The background service (`dockerd`) that actually builds images and runs containers. |

## Installing Docker

- **Windows/Mac**: Install [Docker Desktop](https://www.docker.com/products/docker-desktop/). It bundles the Docker Engine, CLI, and a GUI.
- **Linux**: Install Docker Engine directly via your package manager (no Desktop needed). See Docker's official docs for your distro.

Verify it worked:

```bash
docker --version
docker run hello-world
```

If `hello-world` prints a friendly message, Docker is working: it pulled a tiny test image and ran it as a container.

## Your first real commands

Pull an image (doesn't run it, just downloads it):

```bash
docker pull python:3.11-slim
```

Run a container from an image, interactively:

```bash
docker run -it python:3.11-slim bash
```

- `-it` = interactive + attach a terminal (so you get a shell prompt inside the container)
- Once inside, you're in a totally isolated Linux filesystem with Python 3.11 installed. Try `python3 --version`, mess around, then `exit`.

Run something and immediately remove the container after it exits (good for one-off commands, avoids clutter):

```bash
docker run --rm python:3.11-slim python3 -c "print('hello from inside a container')"
```

List running containers:

```bash
docker ps
```

List **all** containers, including stopped ones:

```bash
docker ps -a
```

Stop a running container:

```bash
docker stop <container_id_or_name>
```

Remove a container:

```bash
docker rm <container_id_or_name>
```

List images you've downloaded/built locally:

```bash
docker images
```

Remove an image:

```bash
docker rmi <image_id_or_name>
```

## Port mapping (so you can actually reach what's running inside)

By default, whatever runs inside a container is invisible to your host machine's network. If you're running a web server on port 8000 inside the container, you need to explicitly map it:

```bash
docker run -p 8000:8000 my-image
```

This means: "host port 8000 → container port 8000". Now `http://localhost:8000` on your machine reaches the server running inside the container.

## Naming containers (so you're not copy-pasting hashes)

```bash
docker run --name my-container -it python:3.11-slim bash
docker stop my-container
docker start -i my-container
```

## A note on Docker Desktop's GUI

Docker Desktop has a GUI showing running containers, images, volumes, and logs. It's genuinely useful for beginners to visually see what's running — use it alongside the CLI, not instead of it. You need the CLI fluency for real work (scripts, CI/CD, servers with no GUI).

## You should now be able to...

- Explain the difference between an image and a container to someone else, without hand-waving
- Explain why containers are lighter than VMs
- Pull an image, run it interactively, exit, and clean it up
- Map a container's port to your host machine
- List, stop, and remove containers and images

## Exercise

1. Pull `python:3.11-slim` and `node:20-slim`.
2. Run each interactively and check the language version inside.
3. Run `docker ps -a` and count how many stopped containers you've accumulated. Remove them all with `docker rm $(docker ps -aq)`.

Next: [Phase 2 — Writing your own Dockerfile](02-phase2-dockerfile.md)
