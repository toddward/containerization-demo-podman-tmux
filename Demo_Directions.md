# Containerization Demo — Build & Run an NGINX Container with Podman

## Overview

In this hands-on demo you will:

1. Confirm Podman is running in **rootless** mode.
2. Start a separate **tmux** session (so multiple users can work on the same shared account without stepping on each other).
3. Set a **`MY_NAME`** environment variable that personalizes every command in the demo.
4. Create a minimal **Containerfile** that builds a custom NGINX image serving a plain-text "I'm up!" page.
5. Build the image with **Podman**.
6. Run the container and verify it responds with `curl` or in a browser.
7. Clean up when you're done.

> **Environment note:** Everyone is logged into a single shared account. Each participant **must** use their own tmux session and a unique container/image name to avoid conflicts. Because we share one account, you will see other participants' images and containers in `podman images` and `podman ps` — this is normal. Only touch resources tagged with **your** name.

**Expected time:** ~15 minutes.

---

## Prerequisites

Before you begin, confirm the following tools are available:

```bash
podman --version
tmux -V
curl --version
```

If any of these are missing, let the instructor know before proceeding.

### Confirm Rootless Mode

This demo runs Podman in **rootless** mode — no `sudo` required. Verify that you are not running as root and that your user has proper subuid/subgid mappings:

```bash
whoami                              # Should NOT be "root"
podman info --format '{{.Host.Security.Rootless}}'  # Should print "true"
grep "^$(whoami):" /etc/subuid      # Should return a mapping for your user
```

If `Rootless` shows `false` or the `subuid` check returns nothing, let the instructor know before proceeding. All commands in this demo run without `sudo`.

> **Why rootless?** Rootless containers run entirely under your normal user account with no elevated privileges. This is a security best practice — even if a container is compromised, the attacker has no root access to the host.

> **Port restriction:** Rootless Podman cannot bind to ports below 1024. We use the `-P` flag (random high port) throughout this demo, so this is handled automatically. Do **not** try `-p 80:80` — it will fail without root.

---

## Instructor Pre-Flight (before participants begin)

```bash
# Pre-pull the base image so participants don't all hit the network at once
podman pull docker.io/library/nginx:alpine

# Verify Podman, tmux, and curl are installed
podman --version && tmux -V && curl --version

# Confirm subuid/subgid mappings exist for the shared account
grep "^$(whoami):" /etc/subuid
```

---

## Step 1 — Start Your tmux Session

Because we are all sharing one account, each participant needs their own separate terminal session. tmux gives you your own terminal — but note that it does **not** isolate your Podman storage or processes from other participants. Unique naming is what keeps everyone's work separate.

First, check for existing sessions to avoid name collisions:

```bash
tmux ls
```

> If you see "no server running on ..." that just means no sessions exist yet — you're the first. That's fine.

Then create your session with a unique name (e.g., your first name or initials):

```bash
tmux new-session -s tward
```

### Helpful tmux Commands

| Action | Keys / Command |
|---|---|
| Detach from session | `Ctrl-b` then `d` |
| Re-attach to your session | `tmux attach -t <session-name>` |
| List all sessions | `tmux ls` |
| Kill your session (cleanup) | `tmux kill-session -t <session-name>` |

---

## Step 2 — Set Your Name Variable

This is the only place you need to type your name. Every command in the rest of this demo uses `$MY_NAME` automatically.

```bash
export MY_NAME="tward"
```

> **Replace `tward` with your own name or initials.** Keep it short, lowercase, and unique among participants (e.g., `jsmith`, `alice`, `mj`).

Verify it is set:

```bash
echo $MY_NAME
```

> **Important:** If you detach and reattach to tmux, or open a new shell, you must run `export MY_NAME="..."` again — environment variables do not survive across sessions.

---

## Step 3 — Create a Project Directory

Set up a working directory for your demo files.

```bash
mkdir -p ~/container-demo-$MY_NAME && cd ~/container-demo-$MY_NAME
```

Verify you are in the correct directory:

```bash
pwd
```

You should see something like `/home/<user>/container-demo-tward`.

---

## Step 4 — Write the Response Page

We will serve a small static file that NGINX returns to every request.

```bash
cat > index.html << EOF
NGINX is up and running!
Container built by: $MY_NAME
EOF
```

> **Note:** This heredoc uses an **unquoted** `EOF` so the shell expands `$MY_NAME` into your actual name.

Verify the file looks correct:

```bash
cat index.html
```

You should see your name in the output — **not** the literal text `$MY_NAME`.

---

## Step 5 — Write the Containerfile

Create the Containerfile (Podman's equivalent of a Dockerfile — Podman accepts both names, but `Containerfile` is the convention).

```bash
cat > Containerfile << 'EOF'
FROM docker.io/library/nginx:alpine

# Remove the default NGINX welcome page
RUN rm /usr/share/nginx/html/index.html

# Copy in our custom response page
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80 inside the container
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
EOF
```

> **Note:** This heredoc uses a **quoted** `'EOF'` because the Containerfile content should be taken literally — no shell expansion needed.

### What's Happening Here

- **`FROM docker.io/library/nginx:alpine`** — Starts from the official NGINX image using the lightweight Alpine variant. We use the fully-qualified registry path (`docker.io/library/`) which is a Podman best practice.
- **`RUN rm ...`** — Removes the default welcome page so ours takes its place.
- **`COPY index.html ...`** — Places our custom page where NGINX serves static files from.
- **`EXPOSE 80`** — Documents the port the container listens on. The `-P` flag in Step 7 uses this to know which ports to publish.
- **`CMD`** — Runs NGINX in the foreground so the container stays alive.

---

## Step 6 — Build the Image

First, confirm you are in your project directory:

```bash
pwd
# Should show the full path, e.g.: /home/<user>/container-demo-tward
ls
# Should show: Containerfile  index.html
```

Build the container image with Podman:

```bash
podman build -t nginx-demo-$MY_NAME .
```

> **First build note:** If the base image has not been pre-pulled, Podman will download `nginx:alpine` (~40 MB). You will see "Trying to pull docker.io/library/nginx:alpine..." — this is normal and may take a minute.

You should see output ending with something like:

```
COMMIT nginx-demo-tward
Successfully tagged localhost/nginx-demo-tward:latest
```

### Verify the Image Exists

```bash
podman images | grep nginx-demo-$MY_NAME
```

If nothing appears, re-run the `podman build` command and check for errors in the output.

---

## Step 7 — Run the Container

Start the container and let Podman assign a random available high port on the host. The `-P` flag publishes every port listed in the `EXPOSE` directive to a random available host port (equivalent to `-p <random>:80`). This avoids port conflicts between participants.

```bash
podman run -d --name mycontainer-$MY_NAME -P nginx-demo-$MY_NAME
```

You will see a long container ID hash printed. Verify the container is actually running:

```bash
podman ps --filter name=mycontainer-$MY_NAME
```

You should see one row with a `STATUS` of `Up`. If the container is not listed, check for errors with `podman logs mycontainer-$MY_NAME`.

### Find Your Assigned Port

Podman mapped a random host port to port 80 inside the container. Find it with:

```bash
podman port mycontainer-$MY_NAME
```

You'll see output like:

```
80/tcp -> 0.0.0.0:43567
```

In this example, your container is reachable on port **43567**.

---

## Step 8 — Verify It Works

### Option A — curl (from the command line)

Using the port number from the previous step:

```bash
curl http://localhost:$(podman port mycontainer-$MY_NAME | cut -d: -f2)
```

> This command automatically extracts your assigned port — no manual substitution needed.

**Expected output:**

```
NGINX is up and running!
Container built by: tward
```

(You will see your own name instead of `tward`.)

### Option B — Browser

First, find the host IP address:

```bash
hostname -I          # Linux
# If that doesn't work, try: ip -4 addr show | grep inet
```

Then open a browser and navigate to:

```
http://<host-ip>:<your-port>
```

You should see the same plain-text message.

---

## Step 9 — Explore (Optional)

Now that your container is running, try some of these to explore further:

```bash
# View running containers (you will see other participants' containers too)
podman ps

# View only YOUR container
podman ps --filter name=mycontainer-$MY_NAME

# Check the container logs
podman logs mycontainer-$MY_NAME

# Exec into the running container (opens a shell inside)
podman exec -it mycontainer-$MY_NAME /bin/sh
# Type 'exit' or press Ctrl-D to leave the container shell

# Inspect the container details
podman inspect mycontainer-$MY_NAME
```

---

## Step 10 — Clean Up

When you're finished, tear everything down so the shared environment stays tidy.

> **Order matters:** You must remove a container before removing the image it was built from. The `-f` flag stops a running container automatically before removing it.

```bash
# Stop and remove your container in one step
podman rm -f mycontainer-$MY_NAME

# Remove your image
podman rmi nginx-demo-$MY_NAME

# Remove your project directory
rm -rf ~/container-demo-$MY_NAME

# Kill your tmux session (this will close your terminal — you're done!)
tmux kill-session -t $(tmux display-message -p '#S')
```

> **Note:** The base `nginx:alpine` image is shared by all participants. The instructor will clean it up after the session.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `echo $MY_NAME` is blank | You need to re-export the variable: `export MY_NAME="yourname"`. This happens if you opened a new shell or reattached to tmux. |
| `podman build` fails with a network error | The base image may not be pre-pulled. Ask the instructor to run `podman pull docker.io/library/nginx:alpine`. Also check internet access. |
| `podman build` can't find the Containerfile | You are likely in the wrong directory. Run `cd ~/container-demo-$MY_NAME` and try again. |
| Port already in use | The `-P` flag should prevent this. If it happens, stop any conflicting container: `podman ps` then `podman stop <id>`. |
| Permission denied on Podman | Podman should be running rootless. Run `podman info --format '{{.Host.Security.Rootless}}'` — if it shows `false`, ask the instructor. Also check: `grep "^$(whoami):" /etc/subuid`. |
| Tried `-p 80:80` and got permission denied | Rootless Podman cannot bind to ports below 1024. Use `-P` for a random high port instead. |
| tmux session name already taken | Run `tmux ls` to see existing sessions. Pick a more unique name. |
| `curl` connection refused | Make sure you're using the correct port from `podman port`. The container may also take a second to start — wait a moment and retry. |
| Disconnected from tmux | Run `tmux attach -t <session-name>` to reconnect. Then run `export MY_NAME="yourname"` and `cd ~/container-demo-$MY_NAME`. |
| Container name already in use | You may have run `podman run` twice. Run `podman rm -f mycontainer-$MY_NAME` and try again. |

---

## Quick Reference

```text
export MY_NAME="yourname"                        # Set once — used everywhere
podman info --format '{{.Host.Security.Rootless}}'  # Confirm rootless mode
tmux new-session -s yourname                     # Start separate terminal session
podman build -t nginx-demo-$MY_NAME .            # Build image from Containerfile
podman run -d --name mycontainer-$MY_NAME -P nginx-demo-$MY_NAME
                                                 # Run container, random high port
podman port mycontainer-$MY_NAME                 # Show port mapping
podman ps --filter name=mycontainer-$MY_NAME     # Show your container status
podman rm -f mycontainer-$MY_NAME                # Stop and remove container
podman rmi nginx-demo-$MY_NAME                   # Remove image
```
