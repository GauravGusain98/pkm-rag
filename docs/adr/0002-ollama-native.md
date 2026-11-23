# ADR-0002: Run Ollama natively on macOS, not in Docker

- **Status:** accepted
- **Date:** 2026-11-23
- **Phase:** 0

## Context
- Docker Desktop's VM can't use the Metal GPU, so in Docker the model runs on the CPU only (GPU → CPU-only → slow).
- I have 8 GB of RAM, and the Docker VM reserves its own slice of it. The model competes with Postgres and Redis inside the VM's memory slice.
- Every RAG answer and every summary job waits on generation.

## Options considered
1. **Option A: Docker Based Ollama**: 
  - pro: one command, pinned, health-checked
  - con: CPU only (see Context)
2. **Option B: Native Ollama Setup**: Installing Ollama outside of Docker is a bit more extra work. But that way we can avoid the major drawback of Option A that is not able to use GPU due to which our RAG response has to wait
3. **Option C: Cloud Based Model**: Using cloud based model is fastest and easier to setup. But this options breaks the goal of personal knowledge base because then the document would be sent to a third party service. This options also prevent our app from working offline and will also be not free as we have to pay per token. 

## Decision
We chose Option B: Native Ollama Setup because it gives us the privacy and offline setup and also allow us to use GPU from our machine and LLM generation is faster

## Consequences
If we run Ollama outside of the docker container then we will need to do some extra work for:

- Health checks: Need to create a GET /health endpoint that reports when Ollama is down. Also we need a process supervisor to restart Ollama if it is down using `brew services start ollama`
- Pinned versions: the model tag is pinned in .env, but the Ollama version isn't. We need to also manage that in Readme
- One-command startup: ollama pull llama3.2:3b is a manual setup step.
- Managing the host.docker.internal connection between docker containers and Ollama running on host machine

This needs to revisit if we think of deploying the project to a linux server with NVIDIA GPU as the docker container then can use this GPU through the NVIDIA Container Toolkit. Another smaller issue is that host.docker.internal is not recognized in Linux servers.

## What I learned
- Docker DNS naming
- Docker Desktop's VM and host.docker.internal
- Docker Container's Loopback Mechanism
