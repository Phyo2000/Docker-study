# Understanding Docker: A Mental Model, Not a Command List

You already know *how* to type `docker compose up`. This guide is about knowing *what happens* when you do — so the commands stop feeling like magic spells.

---

## 0. The One Sentence Version

> **Docker lets you package "my code + everything it needs to run" into a single, portable unit, and run that unit in an isolated mini-environment on any machine.**

Everything else in this guide is just unpacking that sentence.

---

## 1. The Problem Docker Solves

**The situation you've probably lived through:**

> "It works on my computer, but not on yours."

Why does this happen? Because "your computer" is actually a huge pile of invisible state:
- Python 3.11 vs 3.9
- MySQL 8.0 vs 5.7
- A system library that's installed on your OS but not theirs
- An environment variable you set six months ago and forgot about
- A config file sitting in `/etc/` that only exists on your laptop

None of this is written down anywhere. Your project's `.py` files don't capture it. So when your teammate clones the repo, they get *your code* but not *your environment* — and things break in ways that make no sense to them.

**What Docker actually does:** it lets you write your entire environment down as code (a `Dockerfile`), build it into a single artifact (an image), and run that *exact same artifact* everywhere — your laptop, your teammate's laptop, a cloud server. There is no "it works on my machine" anymore, because there is only one machine: the container.

**Analogy:** Think of a shipping container (literally where Docker got its name and logo). Before standardized shipping containers, cargo was loaded by hand, piece by piece, differently for every ship. Standardized containers meant any crane, any ship, any truck could move the exact same box without caring what's inside. Docker does this for software: any machine with Docker installed can run your container without caring what's inside it.

---

## 2. What a Container Actually Is

**What is it?**
A container is a running process (or small group of processes) on your computer that has been given the *illusion* of having its own filesystem, its own network stack, and its own process list — while actually still running directly on your computer's Linux kernel.

**What's happening behind the scenes?**
This is the part most tutorials skip. A container is **not** a mini virtual computer. It's your normal OS kernel, using a few specific Linux features to fence off one process from the rest:

- **Namespaces** — give the process its own *view* of things. A process in a container can see its own `/` filesystem root, its own list of running processes (it thinks it's PID 1!), its own network interfaces — even though the real, full-fat OS underneath sees everything.
- **cgroups (control groups)** — limit how much CPU/memory/disk that process is *allowed* to use, so one container can't starve the whole machine.
- **A layered filesystem (union filesystem)** — stacks read-only "layers" (from the image) with a thin writable layer on top (unique to this container).

So when you "run a container," you are not booting an operating system. You are asking the Linux kernel: *"Start this one process, but put blinders on it so it only sees this filesystem, this network, and these resource limits."*

**Why do I need this?**
Because it gives you isolation (my Django app's Python version can't clash with your Flask app's Python version, even on the same host) *without* the enormous overhead of booting a full separate OS for every app.

**What would happen if I didn't use it?**
You'd install everything directly onto your OS: Python, MySQL, Node, whatever each project needs — all sharing one global environment. Two projects needing different MySQL versions? Good luck. Uninstall one project and something else quietly breaks because they shared a library. This is literally the "dependency hell" that existed before containers were common.

**Real-world analogy:**
Imagine an apartment building. Every tenant (container) has their own locked apartment (isolated filesystem/processes) with their own furniture, but they all share the same building infrastructure — plumbing, electricity, foundation (the host kernel). You don't need to build a separate house (a full VM) for every tenant.

---

## 3. Docker vs a Normal Virtual Machine

This is the comparison that makes "container ≠ mini computer" click.

```text
        VIRTUAL MACHINE                          DOCKER CONTAINER
   ┌────────────────────────┐            ┌────────────────────────┐
   │        App A            │            │        App A            │
   │   Libraries / Bins       │            │   Libraries / Bins       │
   │  Guest OS (full kernel!) │            │  (shares host kernel)   │
   ├────────────────────────┤            ├────────────────────────┤
   │      Hypervisor          │            │     Docker Engine        │
   ├────────────────────────┤            ├────────────────────────┤
   │       Host OS             │            │       Host OS             │
   ├────────────────────────┤            ├────────────────────────┤
   │       Hardware            │            │       Hardware            │
   └────────────────────────┘            └────────────────────────┘

   Boots in: ~30-60 sec                  Starts in: <1 sec
   Size: several GB (full OS)            Size: MBs–hundreds of MBs
   Isolation: very strong (own kernel)   Isolation: process-level (shared kernel)
```

**What is happening behind the scenes?**
A VM emulates entire hardware and boots a *complete* operating system with its own kernel on top of it — that's why it's slow to start and heavy. A container skips all of that: it reuses the host's kernel and just fences off a process. This is *why* containers start almost instantly and are tiny compared to VMs.

**What would happen if you used a VM instead for this use case?**
It would technically work, but you'd pay for it in boot time, disk space, and RAM — for something you're going to spin up and tear down constantly during development. Containers are optimized for exactly that "spin up, test, tear down, repeat" workflow.

**Trade-off to know:** because containers share the host kernel, isolation is slightly weaker than a VM's (a VM has a full hardware+kernel wall between guests). This is why on Windows/Mac, Docker actually runs a lightweight Linux VM in the background — Docker Desktop hides this from you, but there's a small real VM under the hood, because Docker containers ultimately need a Linux kernel to share.

---

## 4. Image vs Container

This trips up almost everyone at first, so nail this analogy:

> **An image is a class. A container is an object (an instance) of that class.**

Or, more concretely:

> **An image is a recipe/blueprint (frozen, read-only). A container is the actual cake/house built from that blueprint (running, has state).**

```text
   Image (my-django-app:latest)
        │
        │  docker run  (create + start a new container from the image)
        ▼
   Container #1  ──┐
   Container #2  ──┼── all built from the SAME image,
   Container #3  ──┘   but each has its own writable layer & state
```

**What is happening behind the scenes?**
An image is a stack of read-only filesystem layers (Python installed, your code copied in, dependencies installed — each `Dockerfile` instruction typically creates a layer). When you run a container from that image, Docker adds one new, empty, *writable* layer on top. Anything the running container writes (log files, temp files, database writes if not using a volume) goes into that writable layer — the image layers underneath are never touched.

**Why does this matter?**
Because it means:
1. You can start 10 containers from 1 image, and they don't interfere with each other's writable layers.
2. Deleting a container does **not** delete the image — you can spin up a fresh, clean container from the same image instantly.
3. It explains why containers are disposable: the container is just "image + a thin scratchpad layer." If a container gets into a weird state, you just delete it and make a new one from the same image — cheap and fast.

**What would happen if I didn't understand this distinction?**
You'd `docker run` five times thinking you're "restarting the same thing," get five different containers piling up, and be confused why old data/logs seem to vanish or duplicate.

---

## 5. Dockerfile

**What is it?**
A text file of step-by-step instructions telling Docker how to *build an image* — essentially "the recipe" mentioned above.

**Practical example (Django app):**
```dockerfile
FROM python:3.11-slim          # start from a base image that already has Python 3.11
WORKDIR /app                   # all following commands run inside /app in the image
COPY requirements.txt .        # copy just this file first
RUN pip install -r requirements.txt   # install dependencies (this becomes a cached layer)
COPY . .                       # now copy the rest of your source code
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]  # default command when container starts
```

**Command → What Docker receives → What Docker does → What I see:**

| You write | Docker does | Why it matters |
|---|---|---|
| `FROM python:3.11-slim` | Downloads/uses a pre-built base image as layer 1 | You don't build Python from scratch — you stand on an existing image |
| `WORKDIR /app` | Sets the working directory for all following instructions | Keeps paths clean and consistent |
| `COPY requirements.txt .` | Adds a new layer containing just that file | Separated from `COPY . .` on purpose (see below) |
| `RUN pip install ...` | Executes the command *at build time*, bakes the result into a new layer | Dependencies become part of the image, not reinstalled every run |
| `COPY . .` | Adds your actual source code as another layer | Comes *after* installing deps |
| `CMD [...]` | Records the default command to run *when a container starts* (not at build time) | This is what actually runs your app |

**Why separate `COPY requirements.txt .` from `COPY . .`?**
Docker caches layers. If you change a `.py` file but not `requirements.txt`, Docker can reuse the cached "install dependencies" layer instead of reinstalling everything from scratch. If you'd written `COPY . .` first, *any* code change would invalidate the cache and force a full reinstall every time you build. This one ordering trick is the difference between a 2-second rebuild and a 2-minute rebuild.

**What would happen if I didn't use a Dockerfile?**
You'd have to manually run commands inside a container every time to install things — nothing would be reproducible, and you couldn't share "how to build this environment" with anyone else or with your future self.

**`RUN` vs `CMD` — a common point of confusion:**
- `RUN` executes **during the build**, and its result is baked into the image forever.
- `CMD` (or `ENTRYPOINT`) is **not executed during build** — it's just recorded as "the thing to run when someone starts a container from this image."

---

## 6. Docker Compose

**What is it?**
A tool + a YAML file (`docker-compose.yml`) for defining and running **multiple containers together** as one coordinated application, instead of typing long `docker run` commands for each one by hand.

**Why do I need it?**
A real app is rarely one container. Your Django app needs a database. Maybe also Redis. Maybe a background worker. Manually running and connecting 3-4 containers with the right networks, volumes, and environment variables every time is unmanageable. Compose lets you describe the *whole system* in one file and bring it all up/down with one command.

**Example `docker-compose.yml` for Django + MySQL:**
```yaml
services:
  web:
    build: .                     # build using the Dockerfile in this folder
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: myapp
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

**What's happening behind the scenes when you define this?**
Compose reads this file and translates it into a set of Docker Engine API calls: build/pull images, create a shared network, create volumes, start containers in dependency order, and wire them together — all the plumbing you'd otherwise have to script by hand with raw `docker` commands.

**What would happen if I didn't use Compose?**
You'd need something like:
```bash
docker network create myapp-net
docker volume create db_data
docker run -d --name db --network myapp-net -e MYSQL_DATABASE=myapp -v db_data:/var/lib/mysql mysql:8.0
docker build -t myapp-web .
docker run -d --name web --network myapp-net -p 8000:8000 -v $(pwd):/app myapp-web
```
...every single time, remembering the exact flags, in the exact order. Compose just remembers this for you as a file.

---

## 7. Volumes

**What is it?**
A mechanism for storing data *outside* a container's writable layer, so it survives even when the container is deleted.

**Why do I need it?**
Remember: a container's writable layer is deleted along with the container. For something like a MySQL database, that's catastrophic — every time you `docker compose down` and back `up`, your data would vanish.

**What's happening behind the scenes?**
A volume is a directory managed by Docker (usually somewhere like `/var/lib/docker/volumes/...` on the host), which gets *mounted* into a specific path inside the container (e.g. `/var/lib/mysql`). The container thinks it's writing to its normal internal filesystem — but that path is actually pointing at persistent storage that lives independently of the container's lifecycle.

```text
   Container (temporary)                Docker Volume (persistent)
   ┌───────────────────┐               ┌────────────────────┐
   │  /var/lib/mysql ───┼──── mounted ──┼──►  actual data files │
   └───────────────────┘               └────────────────────┘
   Delete container → gone              Volume → still here
```

**What would happen if I didn't use one?**
`docker compose down` (or any container deletion) would silently wipe your database. This is the classic "wait, where did all my data go?" moment for beginners.

**Bonus — bind mounts:** in the Compose example above, `- .:/app` for the `web` service is a *bind mount*, not a named volume — it maps your actual project folder on your host machine directly into the container. That's *why* editing a `.py` file on your laptop shows up live inside the running container (great for development, since you don't need to rebuild the image every time you change code).

---

## 8. Networks

**What is it?**
A virtual network that Docker creates so containers can talk to each other **by name**, as if they were separate machines on the same LAN.

**Why do I need it?**
Your Django container needs to reach your MySQL container. They're isolated processes with their own network namespaces — by default they can't see each other at all.

**What's happening behind the scenes?**
When you run `docker compose up`, Compose automatically creates a network (named after your project) and attaches every service in the file to it. Docker also runs an internal DNS server on that network, so `db` (the *service name* from your YAML) resolves to the MySQL container's internal IP address automatically.

That's why your Django `settings.py` can say:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': 'db',      # <-- not localhost! this is the service name
        ...
    }
}
```
`db` isn't a real domain name anywhere on the internet — it only resolves inside this Compose-created network.

**What would happen if I didn't have this?**
You'd have to manually find and hardcode container IP addresses (which change every time containers restart), or expose everything through the host, which defeats a lot of the isolation benefit.

**Common bug this explains:** "Django says `could not connect to host localhost`" — because inside the container, `localhost` refers to *the Django container itself*, not the MySQL container. You must use the service name (`db`), not `localhost`, to reach another container.

---

## 9. Ports

**What is it?**
Port mapping connects a port on your **host machine** to a port **inside a container**, so you (outside Docker) can actually reach a service running inside it.

**Why do I need it?**
Containers are isolated — by default, nothing outside Docker can reach a port a container is listening on, even if the app inside is running fine on port 8000.

**What's happening behind the scenes with `"8000:8000"`?**
```text
HOST : 8000  ────►  Docker's virtual network  ────►  CONTAINER : 8000
(what you type          (proxying/forwarding)         (what Django
 in your browser)                                      actually listens on)
```
The format is `"HOST_PORT:CONTAINER_PORT"`. Docker sets up forwarding rules (via iptables/NAT on Linux) so traffic hitting your machine's port 8000 gets routed into the container's port 8000.

**What would happen if I didn't map a port?**
The container could still run Django perfectly fine internally, but you'd get "connection refused" trying to open `localhost:8000` in your browser — the app is running, just not reachable from outside.

**Common bug this explains:** "port 8000 is already being used" — this means something on your *host* is already bound to 8000 (maybe another container, or a `runserver` you forgot was running locally). Fix: change the host side, e.g. `"8001:8000"` — the container still uses 8000 internally, you just reach it via 8001 from outside.

---

## 10. Environment Variables

**What is it?**
Key-value settings passed into a container at runtime — things like database passwords, debug flags, or API keys.

**Why do I need it?**
You don't want secrets or environment-specific config (like `DEBUG=True` locally vs `DEBUG=False` in production) hardcoded into your source code or your image. The same image should be runnable in different environments just by changing what env vars you feed it.

**What's happening behind the scenes with `.env` + `env_file`?**
Compose reads your `.env` file and injects each key as an environment variable *inside the container's process environment* — the same mechanism as if you'd run `export MYSQL_DATABASE=myapp` before starting the app, just automated. Inside Django, `os.environ.get('MYSQL_DATABASE')` reads this.

```text
.env file  ──(read by Compose)──►  injected as process env vars ──► your Python code reads them
```

**What would happen if I didn't use this?**
You'd either hardcode secrets into your code (bad — especially if it's pushed to GitHub), or you'd need a different image for every environment (dev/staging/prod), defeating the "build once, run anywhere" point of Docker.

**Common bug this explains:** "environment variables are missing" inside the container — usually means the `.env` file isn't actually referenced in `docker-compose.yml` (missing `env_file:`), or the container was started *before* you added the variable and needs a restart to pick it up (env vars are set once, at container start — editing `.env` doesn't affect an already-running container).

---

## 11. Container Lifecycle

```text
   docker create ──► CREATED ──► docker start ──► RUNNING ──► docker stop ──► STOPPED
                                       │                                          │
                                       └──────── docker restart ◄─────────────────┘
                                                                          │
                                                              docker rm (removes it entirely)
```

- **Created** — the container exists (filesystem, config) but hasn't executed its main process yet.
- **Running** — the main process (from `CMD`/`ENTRYPOINT`) is actively executing.
- **Exited/Stopped** — the main process ended (or crashed), but the container (and its writable layer) still exists on disk.
- **Removed** — `docker rm` deletes the container and its writable layer permanently. The image is untouched.

**Key behavior beginners trip over: a container exits the moment its main process ends.**
A container is *not* a persistent VM you can leave "idling" — Docker watches exactly one main process (PID 1 inside the container). The moment that process exits, the container stops, no matter what else might be going on.

**This explains: "my container starts and immediately stops."**
This almost always means the main process crashed or finished instantly — e.g. a misconfigured `CMD`, a script that runs once and exits, or an app that crashed on startup (check with `docker logs <container>` — it will show you the error the process hit right before dying).

---

## 12. What *Actually* Happens When You Run `docker compose up`

Walking through the Django + MySQL example step by step:

```text
$ docker compose up

1. Compose reads docker-compose.yml
2. Compose creates a dedicated network (e.g. "myapp_default")
3. Compose creates/attaches named volumes (e.g. "db_data")
4. For the `db` service:
      - pulls the mysql:8.0 image if not already local
      - creates a container attached to the network, with db_data mounted
      - starts it, injecting MYSQL_DATABASE / MYSQL_ROOT_PASSWORD env vars
5. For the `web` service:
      - since `build: .` is set, builds the image from your Dockerfile
        (reusing cached layers where possible)
      - creates a container attached to the same network
      - mounts your project folder as a bind mount (.:/app)
      - maps host port 8000 → container port 8000
      - injects variables from .env
      - waits for `db` to be *started* (depends_on) — NOT necessarily "ready"!
      - starts the container, which runs CMD: `python manage.py runserver 0.0.0.0:8000`
6. Both containers' logs stream into your terminal, interleaved
```

```text
                 docker compose up
                        │
        ┌───────────────┼───────────────┐
        ▼                                ▼
  build/pull images                create network + volumes
        │                                │
        └───────────────┬───────────────┘
                         ▼
              start containers in dependency order
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      MySQL container          Django container
      (db_data volume)         (bind mount + port 8000)
             └────────── shared network ─────────┘
```

**Important gotcha buried in step 5:** `depends_on` only waits for the MySQL *container* to start, not for MySQL *inside* it to actually finish initializing and accept connections. This is exactly why "Django cannot connect to MySQL" often happens on the *first* `up`, and then works fine on a retry — MySQL just wasn't ready yet when Django tried to connect. (Real fix: healthchecks, or retry logic in your app's startup.)

---

## 13. Realistic Debugging Scenarios (and Why They Happen)

| Symptom | What's actually going on |
|---|---|
| Works on my computer, not on another | Someone's local environment (installed packages, OS libs) differs from what's in the Dockerfile — proof that non-containerized setups aren't truly reproducible |
| A Python package version is different | The image wasn't rebuilt after `requirements.txt` changed, or two people built at different times and got different "latest" base images |
| Django cannot connect to MySQL | Using `localhost` instead of the service name `db`; or MySQL container hasn't finished initializing yet (see `depends_on` gotcha above) |
| Container starts and immediately stops | The main process crashed or exited — check `docker logs <container>`; it is *not* "hanging," it already died |
| Port 8000 already in use | Something else on your host (another container, a local `runserver`) already owns that host port — remap the host side |
| Environment variables missing | `.env` not wired into Compose via `env_file`, or the container was started before the variable existed and needs a restart |
| Data disappears after deleting a container | No volume was attached for that data — it lived only in the container's writable layer, which is gone once the container is |
| `up` works but `run` behaves differently | See the command comparison section below — `run` starts a *new*, isolated one-off container, not the same one `up` is managing |

---

## 14. Commands, Explained as Command → Receives → Does → See → Why

### `docker build`
- **Docker receives:** a path to a Dockerfile + build context (your project folder).
- **Docker does:** executes each instruction top to bottom, creating a cached layer per instruction, and tags the final result as an image.
- **You see:** streamed build output — one block of logs per instruction (or "CACHED" if nothing changed).
- **Why:** to turn your Dockerfile "recipe" into an actual reusable image.

### `docker images`
- **Docker receives:** a request to list local images.
- **Docker does:** reads its local image store.
- **You see:** a table of image names, tags, IDs, sizes.
- **Why:** to check what images exist locally before running/building.

### `docker ps`
- **Docker receives:** a request to list containers (running ones by default; add `-a` for all, including stopped).
- **Docker does:** queries the Docker daemon's container state.
- **You see:** container IDs, names, status ("Up 3 minutes" / "Exited (1) 2 min ago"), and port mappings.
- **Why:** to check what's currently running, and to grab a container name/ID for other commands.

### `docker compose up`
- **Docker receives:** the whole `docker-compose.yml`.
- **Docker does:** builds/pulls images, creates network+volumes, starts *all* defined services together, attaches to their logs.
- **You see:** interleaved logs from every service in your terminal (until you `Ctrl+C`, or run with `-d` to detach).
- **Why:** to bring your *entire multi-container application* up as one unit.

### `docker compose down`
- **Docker receives:** a request to tear down what `up` created.
- **Docker does:** stops and **removes** the containers and the network created by `up`. Named volumes are kept **unless** you add `-v` (which also deletes them — be careful, that deletes your database data too).
- **You see:** logs of each container stopping and being removed.
- **Why:** clean teardown between sessions, without needing to delete images.

### `docker compose ps`
- Same idea as `docker ps`, scoped only to this Compose project's containers. Quick sanity check: "is `db` actually running?"

### `docker compose logs`
- **Docker does:** fetches stored stdout/stderr from a service's container(s).
- **Why:** the #1 debugging command — `docker compose logs web` or add `-f` to follow live, when something's broken and you closed the original terminal.

### `docker compose exec`
- **Docker receives:** a service name + a command to run.
- **Docker does:** runs that command *inside the already-running container* for that service, as an additional process alongside the main one.
- **You see:** the output of that command, or an interactive shell if you run e.g. `docker compose exec web bash`.
- **Why:** to poke around inside a container that's already up — e.g. `docker compose exec web python manage.py migrate`, or opening a shell to inspect files.
- **Requires:** the container must already be running.

### `docker compose run`
- **Docker receives:** a service name + a command.
- **Docker does:** creates a **brand-new, one-off container** from that service's image (not the one `up` is managing), runs the given command, and by default does *not* automatically start the service's other dependencies' *ports* the same way `up` does (though `depends_on` containers still get started).
- **You see:** output of that one-off run; the container is disposable afterward.
- **Why:** for one-time tasks that shouldn't live alongside your main running app — e.g. running a database migration once, or an interactive `python manage.py shell` you don't want tangled with your main `web` container's lifecycle.

### `run` vs `exec` — the key difference
```text
docker compose exec web <cmd>     docker compose run web <cmd>
        │                                  │
        ▼                                  ▼
 uses the EXISTING,               creates a NEW, separate
 already-running "web"            one-off container from
 container                        the "web" image
```
- `exec` = "run this command **inside the container that's already up.**" Fails if the container isn't running.
- `run` = "spin up a **fresh, temporary container** from this service's image and run this command in it." Works even if nothing else is running yet, but it's *not* the same container instance as the one `up` manages (different container ID, its own writable layer, gone after it exits unless you keep it).

### `up` vs `run` — the key difference
- `up` starts (and keeps running) **all services** as defined — this is "run my whole app."
- `run` starts **one service**, as a one-off, usually to execute a specific command — this is "run one specific task using this service's image," and by default doesn't map the ports the way `up` would (so don't expect to hit it in your browser).

---

## 15. Reading a Real Project Layout

```text
myproject/
├── Dockerfile           ← recipe: how to build the Django image
├── docker-compose.yml   ← blueprint: which containers exist, how they connect
├── .env                 ← secrets/config values injected as environment variables
└── app/                 ← your actual Django/Flask source code
```

**What each file is doing, in one line each:**
- `Dockerfile` — turns your source code + dependencies into a single reusable image.
- `docker-compose.yml` — describes the full system: which services exist, how they network together, which ports are exposed, which volumes persist data.
- `.env` — supplies environment-specific values (DB password, debug flag) without hardcoding them into code or the image.
- `app/` — your actual application code, which gets either copied into the image (`COPY . .`) or bind-mounted in for live editing during development.

**How the containers communicate:**
1. Compose creates a private network; every service in the YAML joins it automatically.
2. Services reach each other using their **service name** as a hostname (`db`, `web`, `redis`, etc.) — Docker's internal DNS resolves this.
3. `ports:` mappings are the *only* way traffic from outside Docker (your browser, your host machine) gets into a container.
4. `volumes:` are the only way data outlives a container's own lifecycle.
5. `.env`/`environment:` is how each container gets configured differently without changing the image itself.

---

## 16. The Mental Model to Actually Remember

> **An image is a frozen, reusable environment. A container is one live, disposable instance of that environment. Compose is the conductor that starts multiple containers together and wires them into one private network. Volumes are the only thing that survives when a container dies. Ports are the only doors from outside into a container. Environment variables are how you configure the same image differently in different places.**

If you can explain *that one paragraph* to someone else in your own words, you understand Docker's architecture — not just its commands.

---

## 17. Hands-On Exercise

Using the Django + MySQL setup from this guide:

1. Build and start everything: `docker compose up -d --build`
2. Confirm both containers are running: `docker compose ps`
3. Tail Django's logs: `docker compose logs -f web`
4. Open a shell inside the running web container: `docker compose exec web bash`
5. Inside that shell, run `python manage.py migrate`, then `exit`
6. Break it on purpose: change `HOST: 'db'` to `HOST: 'localhost'` in your Django settings, restart, and watch it fail to connect — then explain to yourself *why*, using what you learned about networks.
7. Fix it back, then delete everything **including volumes**: `docker compose down -v`, bring it back up, and confirm your MySQL data is gone (proving you understand what volumes protect against — and what happens without them).

---

## 18. 10 Questions to Test Real Understanding

1. If you delete a container, does the image it was built from also get deleted? Why or why not?
2. Why does `COPY requirements.txt .` usually come *before* `COPY . .` in a Dockerfile?
3. Inside your Django container, why does connecting to `localhost` for the database fail, while connecting to `db` works?
4. What Linux kernel features make a container different from just "a normal process," and different from a VM?
5. If you run `docker compose down` (without `-v`), what happens to your named volumes? What if you add `-v`?
6. What's the actual difference between what `docker compose exec` and `docker compose run` each do under the hood?
7. Why might `docker compose up` fail to connect Django to MySQL on the very first run, but succeed if you just restart the `web` container?
8. If your container "starts and immediately stops," what does that tell you about the main process, and what command would you use to investigate?
9. What's the difference between a bind mount (`.:/app`) and a named volume (`db_data:/var/lib/mysql`), and why does the Django example use one and MySQL the other?
10. Why can two different projects each use `mysql:8.0`, with different databases and passwords, without conflicting — even on the same laptop?

*(Try answering these from memory before checking back against the guide.)*

---

## 19. Common Beginner Mistakes

- **Editing `.env` and expecting a running container to pick it up instantly** — env vars are set once, at container start; you need to restart (or recreate) the container.
- **Using `localhost` to reach another container** — always use the Docker Compose *service name* instead.
- **Forgetting `-v` deletes volumes too** — `docker compose down -v` will wipe your database data, not just stop containers.
- **Treating a container like a persistent VM** — leaving state only inside a container's writable layer, with no volume, then being surprised it's gone.
- **Not understanding the build cache** — putting `COPY . .` before installing dependencies, causing every code change to trigger a full dependency reinstall.
- **Confusing `run` with `up`** — using `docker compose run web` to "start the app" and being confused why the port isn't reachable in the browser.
- **Assuming `depends_on` means "wait until ready"** — it only waits for the container to *start*, not for the service inside (e.g. MySQL) to finish initializing.
- **Rebuilding without `--build`** — changing the Dockerfile but running `docker compose up` without `--build`, so Compose keeps using the old image.

---

## 20. Cheat Sheet (Reference, Not for Memorizing First)

```bash
# Build & run
docker compose up              # start everything, logs in foreground
docker compose up -d           # start everything, detached (background)
docker compose up -d --build   # rebuild images first, then start detached
docker compose down            # stop & remove containers + network (keeps volumes)
docker compose down -v         # ...and also delete volumes (data loss!)

# Inspecting
docker compose ps              # what's running in this project
docker compose logs -f web     # follow logs for one service
docker ps -a                   # all containers, including stopped ones
docker images                  # all images on this machine

# Running commands inside containers
docker compose exec web bash                  # shell into a RUNNING container
docker compose run --rm web python manage.py migrate   # one-off task in a NEW container

# Building manually
docker build -t myapp .        # build an image from the Dockerfile in this folder
```

The cheat sheet is only useful *after* the mental model — that's why it's last.
