# DevBoard — Advanced (UI + Go + Postgres)

This is the same DevBoard UI as the `master` branch, but now the data comes
from a **real backend** instead of fake in-memory data.

Three pieces talk to each other:

```
browser  →  frontend (React)  →  backend (Go API)  →  database (Postgres)
```

- **frontend** — the React app. It also forwards anything starting with `/api`
  to the backend.
- **backend** — a small Go program that reads and writes the database.
- **database** — Postgres, with some example projects and tasks loaded on first
  start.

There's no login and no AI here on purpose. The whole point is to *see how the
pieces connect*.

---

## Quick start — run it on your own machine

Just cloned this and want to see it working? Pick one of the two ways below.
Both start the same app; they only differ in *what runs it*.

### Option A — Docker Compose (easiest, start here)

Needs **Docker** only.

```bash
git clone https://github.com/sakshi-2028/devboard.git
cd devboard
make up
```

Now open **http://localhost:8080**. You should see the board with example
projects and tasks. To stop it: `make down`.

*(No `make` on your machine? Run `cp .env.example .env` then
`docker compose up --build` instead.)*

### Option B — Kubernetes with kind

Needs **Docker**, **kind** and **kubectl**.

```bash
git clone https://github.com/sakshi-2028/devboard.git
cd devboard

kind create cluster --config kind-config.yaml --name k8s

kubectl apply -f k8s/namespace.yml     # must be first
kubectl apply -f k8s/

kubectl get pods -n devboard -w        # wait for all 3 to say Running, then Ctrl-C
```

The first start takes about **90 seconds** — it has to download the images and
load the example data. For the first minute the pods will look unhappy
(`ContainerCreating`, `Pending`, or a brief `CreateContainerConfigError`). That's
normal, it sorts itself out. Wait for all three to say `1/1 Running`.

Now open **http://localhost:30080**. To stop it:
`kind delete cluster --name k8s`.

You don't need to build anything — the images are already on Docker Hub. Full
explanation, and what to do when something breaks, is in **Part 4** below.

### Which port is which?

Easy to mix up, so keep this handy:

| Where it's running | Open in browser |
| ------------------ | --------------- |
| Docker Compose     | http://localhost:8080 |
| Kubernetes (kind)  | http://localhost:30080 |

---

## What you need

- **Docker** (with Docker Compose, which comes with Docker Desktop).
- That's it for Parts 1–3. You do **not** need Node, Go, or Postgres installed —
  they all run inside containers.
- Part 4 (Kubernetes) additionally needs **kind** and **kubectl**.

---

## Part 1 — The manual way (do it by hand to understand it)

Run all commands from this folder. We'll start the three pieces one by one, the
hard way, so you can see exactly what Docker Compose does for you later.

### Step 1: Create a network

Containers can only find each other by name if they're on the **same network**.
So first we make one:

```bash
docker network create devboard-net
```

### Step 2: Build the images

The frontend and backend are *our* code, so we build an image for each. The
database is not our code — it's the official Postgres image — so there's
nothing to build for it.

```bash
docker build -t devboard-frontend ./frontend
docker build -t devboard-backend ./backend
```

The first build downloads base images and compiles the code, so it can take a
few minutes. Later builds are much faster.

### Step 3: Run the database

We name it `postgres`. The backend will look for it by that exact name. The
`-v ./init/postgres:...` line loads the example data the first time it starts.

```bash
docker run -d --name postgres --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/init/postgres":/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

### Step 4: Run the backend

We name it `backend` (the frontend looks for this name). We also tell it how to
reach the database with `POSTGRES_URL` — notice it uses the name `postgres`.

```bash
docker run -d --name backend --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-backend
```

### Step 5: Run the frontend

It serves the app on port 4173 inside the container; we map it to 8080 on your
machine.

```bash
docker run -d --name frontend --network devboard-net \
  -p 8080:4173 \
  devboard-frontend
```

### Step 6: Open it and check

Open **http://localhost:8080** in your browser — you should see the DevBoard
dashboard with some example tasks. (If the page shows an error for a second on
first load, the backend is still starting up — just refresh.)

Then check the wiring from the terminal:

```bash
curl http://localhost:8081/health                      # backend says OK
curl "http://localhost:8080/api/tasks?project_id=1"    # app → backend → database
```

### Step 7: Stop and clean up

```bash
docker rm -f frontend backend postgres
docker network rm devboard-net
```

### The one thing to remember: names

The backend finds the database using the name `postgres` (see `POSTGRES_URL`).
The frontend finds the backend using the name `backend` (see
`frontend/vite.config.js`). So those container **names must match**, and they
only work because everything is on the same `devboard-net` network.

That's a lot of typing, and you have to start them in the right order. This is
exactly the problem Docker Compose solves.

---

## Part 2 — The easy way: Docker Compose

Compose does everything from Part 1 — the network, the names, the order, the
environment values — from one file (`docker-compose.yml`).

First, create your settings file (one time only). Compose reads it to fill in
passwords and ports, so the stack won't start without it:

```bash
cp .env.example .env
```

Then start everything with one command:

```bash
docker compose up --build
```

The first build can take a few minutes. When it's done, open
**http://localhost:8080** in your browser.

Stop it:

```bash
docker compose down
```

| Piece    | Open in browser / curl        | Notes                                   |
| -------- | ----------------------------- | --------------------------------------- |
| Frontend | http://localhost:8080         | the app; forwards `/api` to the backend |
| Backend  | http://localhost:8081/health  | the Go API (the app uses it via `/api`) |
| Postgres | localhost:5432                | user / password: `devboard` / `devboard`|

---

## Part 3 — The shortcut: `make`

You don't even have to remember the Compose commands. Run `make` to see what's
available:

```bash
make           # list all commands
make setup     # create your .env file (first time only)
make up        # build and start everything
make down      # stop everything
make logs      # watch the logs
make reset     # wipe the database and start fresh
make smoke     # quick check that everything works
```

`make up` creates `.env` for you automatically, so it's the simplest way to start.

> `make` is optional. It's already available on Linux and macOS (on macOS you may
> need Xcode Command Line Tools: `xcode-select --install`). On Windows, either use
> WSL or just run the `docker compose` commands from Part 2 directly.

---

## Part 4 — Kubernetes (kind)

Compose runs the three pieces on one machine. Kubernetes runs the same three
pieces as **pods**, and that's the step where most people get stuck — so follow
these in order.

### What you need for this part

- **kind** — runs a Kubernetes cluster inside Docker (`brew install kind`)
- **kubectl** — the command you talk to the cluster with (`brew install kubectl`)

### Step 1: Create the cluster

Use the `kind-config.yaml` in this folder. **Don't skip it** — see the warning below.

```bash
kind create cluster --config kind-config.yaml --name k8s
```

> **Why the config file matters.** A kind node is just a Docker container. A
> NodePort service is only reachable from your machine if that port is published,
> the same way `-p` works on `docker run`. If you create the cluster with a plain
> `kind create cluster`, then `http://localhost:30080` will refuse to connect —
> even though every pod says `Running` and `1/1 READY`. Nothing warns you.

### Step 2: Get the images (skip this if you haven't changed any code)

The images are already published on Docker Hub, so **you can go straight to
Step 3** — Kubernetes will pull them for you:

```
sakshi2027/devboard-frontend:v1.3
sakshi2027/devboard-backend:v1.0
```

Only do the rest of this step if you've **edited the frontend or backend code**.

A kind cluster cannot see the images on your machine — it has its own separate
image store. So building is not enough; you also have to copy the image in with
`kind load`:

```bash
docker build -t sakshi2027/devboard-frontend:v1.3 ./frontend
docker build -t sakshi2027/devboard-backend:v1.0 ./backend

kind load docker-image sakshi2027/devboard-frontend:v1.3 --name k8s
kind load docker-image sakshi2027/devboard-backend:v1.0 --name k8s
```

Check they arrived:

```bash
docker exec k8s-control-plane crictl images | grep devboard
```

### Step 3: Apply the manifests

Order matters for the first two — the namespace has to exist before anything can
go inside it, and the database should be up before the backend looks for it.

```bash
kubectl apply -f k8s/namespace.yml          # must be first
kubectl apply -f k8s/configMap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/persistent-volume.yml
kubectl apply -f k8s/persistent-volume-claim.yml
kubectl apply -f k8s/postgres-init.yml
kubectl apply -f k8s/postgres-deployment.yml
kubectl apply -f k8s/postgres-service.yml
kubectl apply -f k8s/backend-deployment.yml
kubectl apply -f k8s/backend-service.yml
kubectl apply -f k8s/frontend-deployment.yml
kubectl apply -f k8s/frontend-service.yml
```

> `kubectl apply` never creates a namespace for you. If you skip the first line
> you get `namespaces "devboard" not found`.

### Step 4: Wait for everything, then open it

```bash
kubectl get pods -n devboard -w        # wait until all three say Running, Ctrl-C to stop
```

Then open **http://localhost:30080**.

### Step 5: Check the wiring

```bash
kubectl get all -n devboard
kubectl logs -n devboard deploy/devboard-frontend-deployment
kubectl logs -n devboard deploy/devboard-backend-deployment
```

### Step 6: Clean up

```bash
kubectl delete namespace devboard       # remove the app (everything lives in it)
kind delete cluster --name k8s          # or throw away the whole cluster
```

### If the page loads but there's no data

This is the classic one, and the logs tell you immediately:

```bash
kubectl logs -n devboard deploy/devboard-frontend-deployment
```

If you see `ECONNREFUSED ...:8081`, the frontend is asking for the backend on
the wrong port. **The backend listens on 8080.** Check both `target:` lines in
`frontend/vite.config.js` say `http://backend:8080`.

The `8081` you see in `.env.example` is the *host* port used by Docker Compose so
you can curl the backend directly. Inside the cluster, containers talk to each
other on the **container** port, which is `8080`. Mixing up those two is the most
common cause of an empty board.

### Changed some code? You must rebuild

An image is a frozen snapshot taken at build time. Editing a file on your
machine does **not** change a running pod. Every code change needs the full loop:

```bash
docker build -t sakshi2027/devboard-frontend:v1.4 ./frontend   # 1. new tag
kind load docker-image sakshi2027/devboard-frontend:v1.4 --name k8s   # 2. copy into cluster
# 3. update the image: tag in k8s/frontend-deployment.yml to v1.4
kubectl apply -f k8s/frontend-deployment.yml                   # 4. roll it out
kubectl rollout status deploy/devboard-frontend-deployment -n devboard
```

Always use a **new tag**. If you rebuild with the same tag, the node keeps the
image it already has and nothing changes.

> `frontend-deployment.yml` sets `imagePullPolicy: IfNotPresent`, which means
> "use the local copy if there is one, otherwise pull". That's what lets a
> freshly `kind load`ed image be used without pushing it anywhere. It's safe
> because every build gets its own tag — so a tag's contents never change.

### Common errors

| What you see | What it means |
| ------------ | ------------- |
| `namespaces "devboard" not found` | Apply `namespace.yml` first |
| `ErrImagePull` / `ImagePullBackOff` | Tag isn't on Docker Hub — build it and `kind load` it (Step 2) |
| Pod `Running` but page won't load | Container port and the service's `targetPort` disagree |
| `localhost:30080` refuses to connect | Cluster was made without `kind-config.yaml` — recreate it |
| Page loads, board is empty | Wrong proxy port — see the section above |
| Your code change did nothing | You didn't rebuild + `kind load` + bump the tag |

> **Note on ports.** The frontend container serves on **4173** (that's Vite's
> preview port). `docker-compose.yml` maps `8080:4173`, and in Kubernetes the
> service's `targetPort` is `4173`. If you change the port in
> `frontend/Dockerfile`, you have to change it in **both** of those places too,
> or traffic goes to a port nothing is listening on.

> **Note on `secrets.yml`.** The values there are base64, which is *encoding, not
> encryption* — anyone can decode them in one command. It's fine for practice
> because the password is a throwaway. Real projects use Sealed Secrets, External
> Secrets, or a cloud secret manager.

---

## Settings live in `.env`

All the changeable values (passwords, ports) live in one file. The first time,
copy the example:

```bash
cp .env.example .env     # or: make setup
```

`.env.example` is the template kept in git. Your real `.env` is ignored by git,
so in a real project your secrets never get committed.

---

## The API (for reference)

The browser calls these as `/api/...`; the backend serves them at the root.

| Method | Path                      | What it does                          |
| ------ | ------------------------- | ------------------------------------- |
| GET    | `/projects`               | list projects                         |
| POST   | `/projects`               | create a project                      |
| GET    | `/tasks?project_id=N`     | list tasks in a project               |
| POST   | `/tasks`                  | create a task                         |
| PATCH  | `/tasks/:id`              | update a task (e.g. change status)    |
| GET    | `/search?q=&project_id=N` | search tasks by title                 |
| GET    | `/health`                 | health check                          |

## Folder layout

```
.
├── docker-compose.yml   starts frontend + backend + postgres together
├── Makefile             short commands (make up, make down, ...)
├── .env.example         template for settings (copy to .env)
├── frontend/            React app (Vite). Serves the UI, forwards /api
├── backend/             Go API (main.go + Dockerfile)
├── init/postgres/       schema + example data, loaded on first start
├── kind-config.yaml     kind cluster settings — publishes port 30080
└── k8s/                 Kubernetes manifests (see Part 4)
```
