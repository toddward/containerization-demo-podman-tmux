# Containerization Demo

Materials for a hands-on containerization demo using **Podman** in rootless mode.

## Contents

| File | Description |
|------|-------------|
| `Demo_Directions.md` | Step-by-step participant guide — build and run an NGINX container with Podman |
| `Containerization.pptx` | Slide deck covering containerization concepts |
| `kubeconfig` | Kubernetes configuration file |

## Demo at a Glance

Participants work on a shared Linux account and each build a custom NGINX container that returns a personalized "I'm up!" message. The demo covers:

- Rootless Podman verification
- Writing a `Containerfile`
- Building and tagging an image
- Running a container with automatic port mapping (`-P`)
- Verifying the running service with `curl`
- Cleanup

**Estimated time:** ~15 minutes per participant.

## Prerequisites

- **Podman** (rootless mode with subuid/subgid mappings)
- **tmux**
- **curl**
- Internet access to pull `docker.io/library/nginx:alpine` (or pre-pull before the session)

## Getting Started

Open `Demo_Directions.md` and follow the steps in order. An instructor pre-flight checklist is included at the top of the guide.
