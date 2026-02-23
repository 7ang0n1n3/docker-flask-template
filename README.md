# Flask App Stack

A single `docker-compose.yml` designed for Portainer that clones a Flask app from GitHub, installs its dependencies, runs it under Gunicorn, and automatically pulls updates on a configurable interval — all with no custom image build required.

---

## Contents

- [How It Works](#how-it-works)
- [Requirements](#requirements)
- [Deploying in Portainer](#deploying-in-portainer)
- [Environment Variables](#environment-variables)
- [Authentication](#authentication)
  - [Public Repos](#public-repos)
  - [Private Repo via Token](#private-repo-via-token)
  - [Private Repo via SSH Key](#private-repo-via-ssh-key)
- [Storage Volumes](#storage-volumes)
- [Auto-Update Watcher](#auto-update-watcher)
- [Multiple Flask Apps](#multiple-flask-apps)
- [Flask App Requirements](#flask-app-requirements)
- [Gunicorn Tuning](#gunicorn-tuning)
- [Troubleshooting](#troubleshooting)

---

## How It Works

On every container start, the following steps run in order:

1. **System packages** — installs `git`, `openssh-client`, `ca-certificates`, and `curl` via `apt-get`
2. **Gunicorn** — installed via `uv`
3. **Auth** — configures git credentials (token or SSH key) if the repo is private
4. **Clone or pull** — clones the repo on first start; pulls the latest on subsequent starts
5. **App dependencies** — installs from `pyproject.toml` (editable install) or `requirements.txt`
6. **Storage dirs** — creates the dynamic and static directories if they don't exist
7. **Gunicorn** — starts in the background, bound to `0.0.0.0:<APP_PORT>`
8. **Watcher** — a background loop polls `git fetch` every `UPDATE_INTERVAL` seconds; when a new commit is detected it pulls the changes, reinstalls dependencies, and sends `SIGHUP` to Gunicorn for a graceful reload with zero downtime

The container process waits on Gunicorn and handles `SIGTERM`/`SIGINT` for clean shutdown.

---

## Requirements

- Docker Engine 20.10+
- Docker Compose v2.17+ (bundled with Docker Desktop and recent Docker Engine installs)
- Portainer CE or BE (any recent version)
- A Flask app in a GitHub repository

---

## Deploying in Portainer

1. In Portainer, go to **Stacks → Add Stack**
2. Give the stack a meaningful name (e.g. `myapp`, `blog`, `api`) — this name prefixes the volume names, keeping each app isolated
3. Select **Web editor** and paste the entire contents of `docker-compose.yml`
4. Scroll down to **Environment variables** and add the variables for your app (see table below)
5. Click **Deploy the stack**

To deploy a second app, repeat the steps with a different stack name and different environment variables. Each stack is fully independent.

> **Tip:** You can also store this file in a git repository and point Portainer at it using the **Repository** option instead of pasting. Portainer will pull the compose file from the repo on each redeploy.

---

## Environment Variables

Set these in the Portainer **Environment variables** section when creating or editing the stack.

### Required

| Variable | Description |
|---|---|
| `GITHUB_REPO` | Full clone URL of the Flask app repository (see [Authentication](#authentication) for format) |

### App Configuration

| Variable | Default | Description |
|---|---|---|
| `GITHUB_BRANCH` | `main` | Branch to clone and track for updates |
| `APP_MODULE` | `app:app` | `filename_without_extension:flask_variable_name`. The first part is the `.py` file that contains the Flask object (e.g. `app.py` → `app`, `wsgi.py` → `wsgi`, `wgapp.py` → `wgapp`). The second part is the variable name assigned to `Flask(...)` inside that file. See [Flask App Requirements](#flask-app-requirements) for a full lookup table. |
| `APP_PORT` | `5000` | Port that Gunicorn listens on inside the container, and the host port it maps to |
| `GUNICORN_WORKERS` | `2` | Number of Gunicorn worker processes (see [Gunicorn Tuning](#gunicorn-tuning)) |
| `GUNICORN_THREADS` | `2` | Threads per worker |

### Authentication (private repos only)

| Variable | Default | Description |
|---|---|---|
| `REPO_ACCESS` | `public` | Set to `private` to enable auth |
| `REPO_AUTH_TYPE` | `token` | `token` for a GitHub personal access token, `ssh` for an SSH key |
| `GITHUB_TOKEN` | _(empty)_ | GitHub personal access token — required when `REPO_AUTH_TYPE=token` |
| `SSH_KEY_PATH` | `/run/secrets/ssh_key` | Path to the SSH private key inside the container — required when `REPO_AUTH_TYPE=ssh` |

### Storage

| Variable | Default | Description |
|---|---|---|
| `DYNAMIC_STORAGE` | `/data/dynamic` | Container path for writable runtime data (uploads, SQLite databases, logs, etc.) |
| `STATIC_STORAGE` | `/data/static` | Container path for static assets |

### Auto-Update

| Variable | Default | Description |
|---|---|---|
| `UPDATE_INTERVAL` | `60` | Seconds between `git fetch` checks. Set to `0` to disable (not recommended — use a large number like `86400` instead) |

---

## Authentication

### Public Repos

Set `GITHUB_REPO` to the HTTPS clone URL. No other auth variables are needed.

```
GITHUB_REPO=https://github.com/your-user/your-repo.git
```

### Private Repo via Token

1. In GitHub, go to **Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Create a token with **Contents: Read** permission on the target repository
3. Set the following variables:

```
GITHUB_REPO=https://github.com/your-user/your-repo.git
REPO_ACCESS=private
REPO_AUTH_TYPE=token
GITHUB_TOKEN=github_pat_xxxxxxxxxxxx
```

The token is stored in `~/.git-credentials` inside the container using git's credential store. It is never embedded in the clone URL (which would expose it in process listings).

### Private Repo via SSH Key

1. Generate a deploy key for the repository:
   ```bash
   ssh-keygen -t ed25519 -C "portainer-deploy" -f ./deploy_key -N ""
   ```
2. In GitHub, go to your repo → **Settings → Deploy keys → Add deploy key**
3. Paste the contents of `deploy_key.pub` and enable **Allow read access**
4. Mount the private key into the container by adding a volume entry in the compose file:
   ```yaml
   volumes:
     - app_dynamic:${DYNAMIC_STORAGE:-/data/dynamic}
     - app_static:${STATIC_STORAGE:-/data/static}
     - /path/on/host/deploy_key:/run/secrets/ssh_key:ro
   ```
5. Set the following variables:

```
GITHUB_REPO=git@github.com:your-user/your-repo.git
REPO_ACCESS=private
REPO_AUTH_TYPE=ssh
```

> **Security:** The private key file on the host should be owned by root and mode `600`. The `:ro` mount flag prevents the container from modifying it.

---

## Storage Volumes

The stack creates two named Docker volumes automatically:

| Volume name (Portainer prefix + name) | Default mount point | Purpose |
|---|---|---|
| `<stackname>_app_dynamic` | `/data/dynamic` | Writable runtime data — persists across container restarts and redeployments |
| `<stackname>_app_static` | `/data/static` | Static assets — persists across restarts |

Because Portainer prefixes volume names with the stack name (e.g. `myapp_app_dynamic`), volumes from different stacks never collide.

Your Flask app should read `DYNAMIC_STORAGE` and `STATIC_STORAGE` from the environment to locate these directories:

```python
import os

UPLOAD_FOLDER = os.environ.get("DYNAMIC_STORAGE", "/data/dynamic")
STATIC_FOLDER = os.environ.get("STATIC_STORAGE", "/data/static")
```

To change the mount path (e.g. to `/mnt/uploads`), set `DYNAMIC_STORAGE=/mnt/uploads` — the volume will be mounted at that path and the env var will reflect it.

---

## Auto-Update Watcher

A background process runs inside the container and checks for new commits on the tracked branch every `UPDATE_INTERVAL` seconds.

**What happens when an update is detected:**

1. `git reset --hard origin/<branch>` — applies the new commits
2. `uv pip install` — reinstalls dependencies in case `requirements.txt` or `pyproject.toml` changed
3. `kill -HUP <gunicorn_pid>` — sends `SIGHUP` to the Gunicorn master process, triggering a graceful reload (workers finish their current requests before being replaced by new workers running the updated code)

**Considerations:**

- Database schema migrations are **not** run automatically. If your app requires migrations on deploy (e.g. Flask-Migrate / Alembic), you will need to handle this separately or add a migration step to your app's startup code.
- If a dependency installation fails during the update, Gunicorn is not reloaded — the running app continues serving the previous version.
- To force an immediate update, restart the stack in Portainer. The container will pull the latest code on startup before Gunicorn starts.

---

## Multiple Flask Apps

Each Portainer stack runs one Flask app. To host multiple apps:

1. Deploy this `docker-compose.yml` as a new stack for each app
2. Use a different stack name for each (e.g. `app-blog`, `app-api`, `app-dashboard`)
3. Set a unique `APP_PORT` for each stack so host ports don't conflict

Example port assignments:

| Stack name | APP_PORT |
|---|---|
| `app-blog` | `5000` |
| `app-api` | `5001` |
| `app-dashboard` | `5002` |

Each stack gets its own isolated volumes (`app-blog_app_dynamic`, `app-api_app_dynamic`, etc.) and its own update watcher.

---

## Flask App Requirements

The Flask app repository must meet these requirements for the stack to work correctly:

**Entry point**

`APP_MODULE` follows gunicorn's `module:variable` format:

- **`module`** — the Python file that contains your Flask object, written as its name **without the `.py` extension**. This is NOT a directory or package name, it is a file name.
- **`variable`** — the name of the variable assigned to `Flask(...)` inside that file.

To find the correct values, open the main Python file in your repo and look for the `Flask(` constructor:

```python
# Example: the file is wgapp.py and the object is named wgapp
from flask import Flask
wgapp = Flask(__name__)   # → APP_MODULE=wgapp:wgapp
```

| File in repo root | Flask line | `APP_MODULE` value |
|---|---|---|
| `app.py` | `app = Flask(__name__)` | `app:app` (default) |
| `app.py` | `application = Flask(__name__)` | `app:application` |
| `wsgi.py` | `app = Flask(__name__)` | `wsgi:app` |
| `wgapp.py` | `wgapp = Flask(__name__)` | `wgapp:wgapp` |
| `main.py` | `app = Flask(__name__)` | `main:app` |
| `run.py` | `server = Flask(__name__)` | `run:server` |

> **Common mistake:** setting `APP_MODULE` to the project or package name from `pyproject.toml` instead of the actual `.py` filename. The `name` field in `pyproject.toml` is irrelevant — only the filename and variable name matter.

**Dependencies**

Include either a `requirements.txt` or a `pyproject.toml` at the repo root. The stack auto-detects which one to use. If neither is present, only Flask itself is installed as a fallback.

```
# requirements.txt example
flask>=3.0
flask-sqlalchemy
gunicorn
```

> `gunicorn` is already installed at the system level by the stack, so it does not need to be in `requirements.txt` — but including it there is harmless.

---

## Gunicorn Tuning

The recommended formula for `GUNICORN_WORKERS` is:

```
workers = (2 × CPU cores) + 1
```

For a 2-core host: `GUNICORN_WORKERS=5`
For a 4-core host: `GUNICORN_WORKERS=9`

`GUNICORN_THREADS` controls threads per worker. For I/O-bound apps (APIs, database queries), increasing threads can improve throughput without increasing memory usage significantly. A value of `2`–`4` is a reasonable starting point.

| Use case | Suggested workers | Suggested threads |
|---|---|---|
| Low traffic / home lab | `2` | `2` |
| Medium traffic | `(2 × cores) + 1` | `2` |
| I/O-heavy app | `2–4` | `4–8` |

---

## Troubleshooting

**Container exits immediately on start**

Check the container logs in Portainer (Containers → select container → Logs). Common causes:
- `GITHUB_REPO` is not set
- `GITHUB_TOKEN` is missing when `REPO_ACCESS=private` and `REPO_AUTH_TYPE=token`
- SSH key file not found at `SSH_KEY_PATH`
- `APP_MODULE` points to a module or variable that doesn't exist in the cloned repo

**App is not found / 502 from reverse proxy**

- Confirm `APP_PORT` matches the port your reverse proxy is forwarding to
- Check that Gunicorn started successfully in the logs — look for `[gunicorn] Starting on 0.0.0.0:<port>`
- Verify `APP_MODULE` matches your Flask app's file and variable name

**Updates are not being applied**

- Check the watcher logs in the container output — look for `[watcher]` lines
- Confirm the container has outbound internet access to reach GitHub
- If using a private repo, ensure the token or SSH key has read access to the repository
- The watcher only reloads Gunicorn when the git commit hash changes — make sure you are pushing to the branch set in `GITHUB_BRANCH`

**Port conflict when deploying multiple apps**

Each stack must use a unique `APP_PORT`. Check Portainer's container list for which ports are already in use before deploying a new stack.

**"Permission denied" on SSH key**

The SSH key file must be readable by root inside the container. On the host, ensure the file mode is `600`:
```bash
chmod 600 /path/to/deploy_key
```
