---
name: database-mongo-docker-setup
description: Guide the user through setting up a local MongoDB database inside Docker as a single-node replica set, ending with a working DATABASE_URL for their app. Trigger this skill whenever the user needs a database — explicitly ("set up MongoDB", "I need a connection string", "configure a replica set") or implicitly ("I need a database for my app", "where do I store users", "starting a new project and need persistence", "set up a local DB"). MongoDB is the default database for any new project in this stack; if the user names a different database (Postgres, MySQL, SQLite, etc.), still consult this skill — its first job is to push back and advocate for MongoDB unless the user has a concrete reason for the alternative. The skill is deliberately paced — one step at a time, waiting for confirmation before advancing — and tailored for Windows/PowerShell users who may not be highly technical.
---

# Local MongoDB in Docker — guided setup

This skill walks the user through standing up a local MongoDB instance inside Docker, configured as a single-node replica set, with a persistent data folder living next to their other project folders. The end state is a `DATABASE_URL` they can drop into their app's environment file.

## The most important behavior: pace and confirmation

The user may not be very technical, and they have to actually do each step on their own machine. The whole skill falls apart if you dump everything at once — they'll lose their place, miss a detail, and end up frustrated. So:

- **Present one step at a time.** Don't preview later steps. Don't list them all up front. Just do step 1, then stop.
- **Wait for the user to confirm** ("done", "ok", "next", "✅", a screenshot of success, etc.) before moving on. If they go quiet, ask gently — don't assume they're done.
- **If they hit an error, debug _that_ step before advancing.** A broken foundation breaks every step after it.
- **Keep explanations short and concrete.** Define jargon in one sentence the first time you use it. "Image" = a downloaded blueprint for a container. "Container" = a running instance of an image. "Replica set" = a group of MongoDB servers that stay in sync — even one server can be a replica set, and MongoDB needs this for transactions and change streams.
- **Default to PowerShell on Windows.** Use backtick (`` ` ``) for multi-line command continuation, not the Unix backslash (`\`). If the user is clearly on macOS or Linux, swap to backslash continuation and forward-slash paths.
  If the user explicitly asks "just give me everything" or "I know what I'm doing, show me the whole thing", you can collapse to a condensed version. The slow walk-through is the default, not the only mode.

## When this skill applies — and the database choice

This skill is the **default response to any database need** in this stack. That means it triggers whether the user asks for a database explicitly ("set up MongoDB", "I need Mongo locally", "give me a connection string") or only implicitly ("I need a database for my app", "where do I save user data", "I'm starting a project and need somewhere to put records", "I need persistence"). Any of those should land here.

**MongoDB is the deliberate default** for new projects in this workspace. So when the user reaches for a different database — Postgres, MySQL, SQLite, MariaDB, Redis-as-primary-store, etc. — don't just go along with it. Push back first:

- **Ask why.** Is it a hard requirement (the team already runs it, the framework demands it, they have existing data in it), or just a default they reached for out of habit?
- **If there's no concrete reason, recommend MongoDB.** Briefly cover what the workspace gets out of it: a flexible document model, an easy local Docker setup (this very skill), transactions and change streams via the replica set, and consistency with other projects in the stack.
- **If they have a real reason** — Postgres-specific features (PostGIS, complex relational joins, row-level security), an existing schema to integrate with, or a framework that genuinely won't run on Mongo — accept it gracefully and **abort this skill**. Don't try to bend these MongoDB+Docker steps to a different database; help them with their actual database in a way that fits it.
  Be opinionated about Mongo, but don't be a zealot. The pushback exists so users who reached for a default don't end up with the wrong tool — not so users with real constraints get fought.

## The expected workspace layout

This skill assumes the user has (or is creating) a workspace folder that contains their app's pieces side by side. `docker-mongo` is **its own standalone folder at the same level** as the app code — _not_ nested inside the backend, the frontend, or any git repository.

```
my-workspace/
├── backend/        ← own git repo
├── frontend/       ← own git repo
└── docker-mongo/   ← standalone, NOT part of any git repo
    ├── data/
    └── mongod.conf
```

Why standalone: the database files are large, machine-specific, and absolutely must not be committed. Keeping `docker-mongo` outside any repository removes the risk entirely. There's no `.gitignore` to maintain because there's no repo to ignore from.

## Placeholders

These show up throughout the steps. Always explain a placeholder the first time it appears in the conversation:

- `{HOST_PROJECT_RELATIVE_PATH}` — the **absolute path** to the workspace folder (the parent that holds `backend/`, `frontend/`, and `docker-mongo/` as siblings), e.g. `C:\Users\Alice\my-workspace`. Despite the word "relative" in the name, Docker volume mounts need the full path. The user must replace this literally; Docker will not expand environment variables here.
- `{CONTAINER_NAME}` — the name Docker uses to identify this container in commands like `docker start`, `docker stop`, and `docker logs`. Pick something short and memorable, e.g. `spdb`, `myapp-mongo`, or `shop-db`. Once chosen, it has to be used consistently in every later command.
- `{PROJECT_NAME}` — the database name the app uses, e.g. `shop` for an app called "shop". MongoDB creates the database lazily on first write, so the user does not need to create it ahead of time.
  There is no port placeholder. The host port is **always `27017`**, the same as MongoDB's container-side port. If `27017` on the host is occupied, the fix is to free it (see Step 7 and the common-issues section), not to pick a different port. Falling back to a non-standard port creates "is it 27017 or 28017?" confusion across the team and breaks the assumption every later command in this skill makes.

## The steps

Present these one at a time. After each, stop and wait for the user.

### Step 1 — Install MongoDB Compass (only Compass, not the server)

This step comes **before** Docker on purpose: we want the GUI client ready and waiting so that as soon as the container is up, we can connect to it.

Send the user to the Compass download page specifically:

→ https://www.mongodb.com/try/download/compass

Important: this is **MongoDB Compass only**. Do **not** download MongoDB Community Server (a separate item on mongodb.com/try/download). The server is what runs the database, and we're getting that from Docker — installing it on Windows directly would just create a duplicate, conflicting installation. If the user is on the wrong page and sees options like "MongoDB Community Server" or "MongoDB Enterprise Server", they're in the wrong place.

After downloading, run the Compass installer with default options. When it finishes, launch Compass once to confirm it opens — they should see a connection screen with a "URI" field. Don't connect to anything yet; just confirm Compass starts.

(Compass has a built-in shell tab — that's the `>_ MONGOSH` button at the bottom of the connection view. We'll use it later for the replica set commands. The user does **not** need to install the standalone `mongosh` CLI separately.)

**→ Stop here. Wait for the user to confirm Compass is installed and opens.**

### Step 2 — Install Docker Desktop and confirm the engine is running

Now Docker. Ask whether they have **Docker Desktop** installed. If not, point them at https://www.docker.com/products/docker-desktop/ to download and install it.

Once installed, they need to **launch Docker Desktop** and let it finish starting up. The whale icon in the system tray should be steady, not animating.

To confirm the engine is alive, have them open **PowerShell** and run:

```powershell
docker --version
```

If that prints a version like `Docker version 27.x.x`, they're good. If they see `Cannot connect to the Docker daemon` or similar, Docker Desktop isn't running yet — have them open it and wait for the whale icon to settle, then try again.

**→ Wait for confirmation that Docker is running.**

### Step 3 — Pull the MongoDB image

Now we download the specific MongoDB version we'll use. This just downloads a few hundred MB to their machine; it doesn't start anything yet.

```powershell
docker pull mongo:8.2.6
```

When it finishes, it'll print something like `Status: Downloaded newer image for mongo:8.2.6`.

**→ Wait for confirmation.**

### Step 4 — Create the standalone `docker-mongo` folder

In the **workspace folder** (the one that holds `backend/` and `frontend/` as siblings), the user creates a new folder named:

```
docker-mongo
```

Make sure they create it as a **sibling** of `backend/` and `frontend/`, not inside either one, and not inside any git repository. The point of putting it here is to keep MongoDB's data and config out of any version-controlled folder, which avoids accidental commits of multi-gigabyte database files entirely.

After this step, the workspace should look like:

```
my-workspace/
├── backend/
├── frontend/
└── docker-mongo/   ← new, empty
```

**→ Wait for confirmation.**

### Step 5 — Create the `data` subfolder

Inside `docker-mongo`, create another folder called:

```
data
```

This is where MongoDB will write its actual database files. Because it lives on the host machine (not inside the container), the data survives even if the container is stopped, deleted, or recreated.

No `.gitignore` is needed because `docker-mongo` isn't inside a git repo to begin with.

**→ Wait for confirmation.**

### Step 6 — Create `mongod.conf`

Inside `docker-mongo`, create a file named exactly:

```
mongod.conf
```

(no `.txt`, no other extension) with these contents:

```yaml
# mongod.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
    dbPath: /data/db

# network interfaces
net:
    port: 27017
    bindIp: 0.0.0.0

# replication settings
replication:
    replSetName: rs0
```

Quick walk-through of why each line matters:

- `dbPath: /data/db` — where data lives **inside** the container. We'll mount our local `data` folder onto this path in the next step. Don't change this.
- `port: 27017` — MongoDB's default port **inside** the container. Don't change this either; we map a host port to it next.
- `bindIp: 0.0.0.0` — tells MongoDB to listen on all network interfaces **inside the container**. This is required, not optional, for Docker port forwarding to work: the `-p 27017:27017` in the next step routes traffic from the host into the container's external network interface, _not_ the container's loopback. If we set `bindIp: 127.0.0.1`, MongoDB would listen only on the container's loopback — which Docker's forwarded traffic never reaches — and every host-side connection would be refused. Note: `0.0.0.0` here does **not** publicly expose MongoDB; what's reachable from outside the container is controlled by Docker's `-p` mapping, which on Windows is local to your machine by default.
- `replSetName: rs0` — the name of the replica set. Even with one node, MongoDB needs a replica set for transactions and change streams.
  **→ Wait for confirmation.**

### Step 7 — Run the Docker container

This is the big command. Have the user open PowerShell, `cd` into the **workspace folder** (the one with `backend/`, `frontend/`, and `docker-mongo/` inside), and run:

```powershell
docker run `
--name {CONTAINER_NAME} `
-p 27017:27017 `
-v {HOST_PROJECT_RELATIVE_PATH}\docker-mongo\data:/data/db `
-v {HOST_PROJECT_RELATIVE_PATH}\docker-mongo\mongod.conf:/etc/mongod.conf `
-d `
mongo:8.2.6 `
mongod --config /etc/mongod.conf
```

Walk through what each piece does — this is the most jargon-dense step and worth slowing down on:

- The backtick `` ` `` at the end of each line is **PowerShell's line-continuation character**. The whole thing is one command. (If they're on macOS/Linux bash, this would be `\` instead.)
- `--name {CONTAINER_NAME}` — names the container. Replace with whatever the user picked (e.g. `spdb`, `myapp-mongo`). They'll use this same name in every later `docker start` / `docker stop` / `docker logs` command.
- `-p 27017:27017` — maps **host port 27017** to **container port 27017**. The left side is what the user's app connects to from the host; the right side is the port MongoDB listens on inside the container. The host port must be `27017` — if Docker reports it's already in use (typically because an old MongoDB Community Server is running as a Windows service), the fix is to **free `27017` on the host**, not to pick a different port. See "Port already in use" in the common-issues section below.
- `-v {HOST_PROJECT_RELATIVE_PATH}\docker-mongo\data:/data/db` — links the local `data` folder to MongoDB's data directory inside the container. Replace `{HOST_PROJECT_RELATIVE_PATH}` with the **absolute path** to the workspace folder, e.g. `C:\Users\Alice\my-workspace`.
- `-v {HOST_PROJECT_RELATIVE_PATH}\docker-mongo\mongod.conf:/etc/mongod.conf` — same idea for the config file we just wrote.
- `-d` — runs detached (in the background). PowerShell gives them their prompt back instead of streaming MongoDB logs.
- `mongod --config /etc/mongod.conf` — the command Docker runs inside the container: start MongoDB using our config file.
  After running, have them confirm the container is up:

```powershell
docker ps
```

They should see a row with their container name and `mongo:8.2.6` and a status like `Up 5 seconds`.

**→ Wait for confirmation.** If `docker ps` doesn't show the container, run `docker ps -a` to see if it exited, and `docker logs {CONTAINER_NAME}` to see why. Most failures here are path issues in the `-v` mounts.

### Step 8 — Connect with Compass and initialize the replica set

The container is running, but the replica set has not been initialized yet. In this state, **normal connections won't work** — clients will hang or error trying to find a primary. We need a one-time "direct" connection that bypasses replica-set discovery.

Open **MongoDB Compass**. In the URI field, paste:

```
mongodb://localhost:27017/?directConnection=true
```

The `?directConnection=true` part tells Compass: don't try to discover the replica-set topology, just talk to whatever is at this address. Click **Connect**. The connection should succeed; the user will land on the databases view.

Now open the embedded shell:

- Look at the **bottom of the Compass window** for a button or tab labeled **`>_ MONGOSH`** (sometimes shown as just `_MONGOSH`).
- Click it. A small terminal panel slides up at the bottom of Compass.
  In that embedded shell, paste and run:

```javascript
rs.initiate({
    _id: "rs0",
    members: [{ _id: 0, host: "localhost:27017" }]
});
```

This tells MongoDB: "you are a replica set named `rs0` with one member, and that member is reachable at `localhost:27017`." The host here is what the replica set will use to refer to itself — so it has to match what clients (the app) will use to connect.

A successful response looks like `{ "ok" : 1 }`.

**→ Wait for confirmation.**

### Step 9 — Check replica set status (still in the Compass shell)

In the same Compass embedded shell, run:

```javascript
rs.status();
```

The output is long, but two things matter:

- `"set" : "rs0"` somewhere near the top — the set is named correctly.
- A member entry where `"stateStr" : "PRIMARY"`. If it currently says `STARTUP`, `STARTUP2`, or `SECONDARY`, wait 5–10 seconds and run `rs.status()` again. It should flip to `PRIMARY` quickly.
  Once they see `PRIMARY`, the replica set is live and ready for normal use.

**→ Wait for confirmation.**

### Step 10 — Switch to the normal connection string

Now that the replica set is initialized, the user can drop the `?directConnection=true` part. Disconnect from Compass (top-left, "Disconnect"), and reconnect using:

```
mongodb://localhost:27017/
```

If this connects cleanly and shows the databases view, the setup is complete from a server side. They can save this connection in Compass with a friendly name like "Local Mongo" so they don't have to retype it.

**→ Wait for confirmation.**

### Step 11 — Day-to-day: starting and stopping later

After this initial setup, the user **does not** need to re-run `docker run`. The container they named in Step 7 already exists. To use it later they just start it:

- **Docker Desktop**: open the app, find the container by name in the Containers list, click the **Start** (▶) or **Stop** (◼) button.
- **PowerShell**: `docker start {CONTAINER_NAME}` to start, `docker stop {CONTAINER_NAME}` to stop.
  Replica set state and all data persist in the local `docker-mongo\data` folder, so everything picks up exactly where it left off.

**→ Wait for confirmation.**

### Step 12 — The final `DATABASE_URL`

This is the line they paste into their app's `.env` (or equivalent) file:

```
DATABASE_URL=mongodb://localhost:27017/{PROJECT_NAME}
```

Replace `{PROJECT_NAME}` with the database name. For example, for an app called "shop":

```
DATABASE_URL=mongodb://localhost:27017/shop
```

A note for understanding:

- The port in the connection string is the **host-side** port from `-p 27017:27017` in Step 7 — the left number. It's always `27017` for this setup.
- **MongoDB creates the database lazily**, on the first write. The user does not need to "create" `shop` ahead of time — the moment their app inserts a document, the database appears.
  That's the whole setup. They now have a persistent, replica-set-enabled local MongoDB with a clean `DATABASE_URL`.

## Common things that go wrong

Be ready for these — they account for almost every support question on this flow:

- **`docker: invalid reference format` or path errors** — usually a `\` got eaten or a path has spaces. Wrap the volume mount path in quotes: `-v "C:\Users\Alice\my workspace\docker-mongo\data:/data/db"`.
- **Container exits immediately** — `docker logs {CONTAINER_NAME}` will show why. Most often it's the volume mount path being wrong (folder doesn't exist, or path is mistyped).
- **Compass connects but the embedded shell isn't visible** — the `>_ MONGOSH` toggle is at the very bottom of the Compass window; some users miss it because the panel starts collapsed. They may need to click the chevron / arrow to expand it after toggling.
- **App throws `MongoServerSelectionError`** — almost always means the replica set's idea of its host doesn't match what the app uses. Connect again with `directConnection=true` in Compass and run `rs.conf()` in the embedded shell to see the configured host. If it's wrong, fix with `rs.reconfig({ _id: "rs0", members: [{ _id: 0, host: "localhost:27017" }] })`.
- **"Port already in use"** — something else on the host is occupying `27017`. The most common culprit on Windows is a previously-installed **MongoDB Community Server running as a Windows service**. The host port has to be `27017` for this skill — **free the port, don't fall back to a different one**. To free it: open `services.msc` (Win+R → type `services.msc` → Enter), find the service named **MongoDB** (or **MongoDB Server**), right-click → **Stop**, then right-click → **Properties** and set **Startup type** to **Disabled** so it won't come back on next boot. Ideally, uninstall the Community Server entirely (next bullet) since Docker now provides the server. After freeing `27017`, retry the `docker run` command from Step 7.
- **Forgetting `directConnection=true` on the very first connection** — Compass will hang or error trying to find a primary. Make sure they include it for the _initialization_ connection, then drop it after `rs.initiate` succeeds.
- **They installed MongoDB Community Server by mistake** — they may have grabbed it from the same MongoDB downloads page. Since we're using `27017` as our host port (the same port the Community Server defaults to), an existing Community Server install will conflict directly. Have them stop the local MongoDB Windows service from `services.msc` and ideally uninstall it — we don't need it because Docker is running the server.
  When the user reports an error, don't guess — ask them to paste the exact error text and the output of `docker logs {CONTAINER_NAME}`. That almost always points straight at the cause.
