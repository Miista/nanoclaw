# Migration Guide: Mac → Raspberry Pi (or any Linux server)

## Prerequisites

- SSH key-based access to the target machine
- Docker installed on target (`curl -fsSL https://get.docker.com | sh && sudo usermod -aG docker $USER`)
- Build tools on target (`sudo apt-get install -y build-essential python3`)
- Same architecture (this setup: Apple Silicon Mac → ARM64 Pi — no cross-platform issues)

**Important:** Don't stop the source machine's containers until you've verified the target is fully working. You may need to go back and re-export data.

---

## Part 1: Docker containers (docker-compose stack)

### On the source machine

1. **Stop any services that write to disk** (optional but safer):
   ```bash
   docker compose stop
   ```

2. **Identify what needs copying:**
   - `docker-compose.yml` and `.env`
   - All bind-mounted host directories (check `volumes:` in compose file for host paths like `/Users/you/linkding-data:/etc/...`)
   - Named volumes require a different approach (see below)

3. **Transfer bind-mount data:**
   ```bash
   rsync -av ~/linkding-data ~/piper-amy-data ~/homeassistant ~/familyscreen-data admin@target:/home/admin/docker/
   ```

4. **Update `docker-compose.yml` paths** — replace all source paths with target paths:
   - `/Users/sorenguldmund/` → `/home/admin/docker/`
   - `build: /path/to/source` → `image: image-name:tag` (if using a pre-built image)

5. **Copy the updated compose and env:**
   ```bash
   rsync -av docker-compose.yml .env admin@target:/home/admin/docker/
   ```

6. **For locally built images** (like `family-dashboard`):
   ```bash
   docker save image-name:tag | ssh admin@target 'docker load'
   ```

7. **Start on target:**
   ```bash
   ssh admin@target "cd ~/docker && docker compose up -d"
   ```

### Named volumes (e.g. OneCLI postgres)

Named volumes can't be rsynced directly. Three approaches:

**Option A — pg_dump (for postgres, recommended):**
```bash
# Dump to file (architecture-independent, cleanest approach)
docker exec onecli-postgres-1 pg_dump -U onecli onecli > ~/docker/onecli-dump.sql

# Transfer to target
rsync -av ~/docker/onecli-dump.sql admin@target:/home/admin/docker/
```

To restore on the target (after starting an empty postgres container):
```bash
cat ~/docker/onecli-dump.sql | docker exec -i onecli-postgres-1 psql -U onecli -d onecli
```

> **Gotcha:** If the target already has data (e.g. from a previous failed attempt), pg_dump will produce duplicate key errors. Drop and recreate the database first, or ignore the errors if the data was already there.

**Option B — tar pipe (works for any volume):**
```bash
# Extract locally
mkdir -p ~/docker/onecli-app-data
docker run --rm -v onecli_app-data:/data alpine tar czf - -C /data . | \
  tar xzf - -C ~/docker/onecli-app-data

# Or pipe directly to target
docker run --rm -v onecli_app-data:/data alpine tar czf - -C /data . | \
  ssh admin@target "mkdir -p ~/docker/onecli-app-data && tar xzf - -C ~/docker/onecli-app-data"
```

**Option C — copy from volume mountpoint (needs sudo on Linux):**
```bash
mkdir -p ~/docker
sudo cp -a $(docker volume inspect onecli_app-data --format '{{.Mountpoint}}') ~/docker/onecli-app-data
sudo cp -a $(docker volume inspect onecli_pgdata --format '{{.Mountpoint}}') ~/docker/onecli-pgdata
sudo chown -R $(whoami) ~/docker/onecli-app-data ~/docker/onecli-pgdata
rsync -av ~/docker/onecli-app-data ~/docker/onecli-pgdata admin@target:/home/admin/docker/
```

---

## Part 2: OneCLI migration

OneCLI has three pieces that must all be migrated together:

| Piece | Location | How to migrate |
|---|---|---|
| Config | `~/.onecli/` | `rsync -av ~/.onecli/ admin@target:/home/admin/.onecli/` |
| App data (incl. encryption key) | Named volume `onecli_app-data` | tar pipe or copy from mountpoint |
| Database | Named volume `onecli_pgdata` | pg_dump (recommended) or copy from mountpoint |

**Critical:** The `secret-encryption-key` file in the app-data volume encrypts secrets stored in the postgres database. If you restore one without the other (or regenerate either), secrets become unreadable. Always migrate them together.

**Critical files in `onecli-app-data/`:**
- `secret-encryption-key` — decrypts stored API keys (must match database)
- `gateway/ca.key` and `gateway/ca.pem` — MITM proxy CA for injecting credentials into API calls
- `runtime-config.json`

### Compose file on target

Convert named volumes to bind mounts pointing to `~/docker/onecli-*` and remove the `volumes:` declarations at the bottom:

```yaml
services:
  postgres:
    volumes:
      - /home/admin/docker/onecli-pgdata:/var/lib/postgresql

  app:
    volumes:
      - /home/admin/docker/onecli-app-data:/app/data
    ports:
      - "${ONECLI_BIND_HOST:-127.0.0.1}:10254:10254"
      - "${ONECLI_BIND_HOST:-127.0.0.1}:10255:10255"
```

### Linux-specific: bind to all interfaces

On Linux, containers reach the host via `172.17.0.1` (the Docker bridge), not `127.0.0.1`. The OneCLI gateway must listen on all interfaces. Create `~/.onecli/.env`:

```
ONECLI_BIND_HOST=0.0.0.0
```

> **Why:** macOS Docker Desktop transparently maps `host.docker.internal` to the host loopback. Linux Docker maps it to the bridge IP (`172.17.0.1`), so the gateway must be reachable there. Without this, agent containers get `502 Bad Gateway` or `connection refused` when trying to call the Anthropic API through the proxy.

### Install OneCLI CLI tool

```bash
curl -fsSL onecli.sh/cli/install | sh
source ~/.bash_profile  # or open new terminal
onecli config set api-host http://127.0.0.1:10254
```

### Start and verify

```bash
docker compose -f ~/.onecli/docker-compose.yml up -d

# If using pg_dump, restore after postgres is healthy:
docker compose -f ~/.onecli/docker-compose.yml ps  # wait for healthy
cat ~/docker/onecli-dump.sql | docker exec -i onecli-postgres-1 psql -U onecli -d onecli

# Verify secrets
onecli secrets list
```

You should see your Anthropic secret. If you get a decrypt error, the `secret-encryption-key` doesn't match the database — re-register with `onecli secrets create`.

You can also verify directly in postgres:
```bash
docker exec onecli-postgres-1 psql -U onecli -d onecli -c 'SELECT name, type FROM secrets;'
```

---

## Part 3: NanoClaw migration

NanoClaw is code + stateful data. The code lives in your GitHub fork; data is local.

### What to copy

| Source | Contains | Gitignored? |
|--------|----------|-------------|
| `store/messages.db` | Message history | Yes |
| `groups/*/` | Per-group memory, CLAUDE.md, conversations, attachments | Yes (except main/global CLAUDE.md) |
| `data/certs/` | OneCLI proxy CA cert (auto-refreshed on start, no need to copy) | Yes |
| `data/env/` | Container env file | Yes |
| `data/ipc/` | IPC fifos (transient, skip) | Yes |
| `data/sessions/` | Agent session data | Yes |
| `.env` | Credentials (TZ, ONECLI_URL, channel tokens) | Yes |
| `logs/` | Log files (optional) | Yes |

### On source

1. **Stop the service:**
   - macOS: `launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist`
   - Linux: `systemctl --user stop nanoclaw`

2. **Transfer stateful data** (exclude node_modules and dist — these must be rebuilt):
   ```bash
   rsync -av --exclude node_modules --exclude dist --exclude 'data/ipc' \
     ~/Developer/nanoclaw/store \
     ~/Developer/nanoclaw/groups \
     ~/Developer/nanoclaw/data \
     ~/Developer/nanoclaw/.env \
     admin@target:/home/admin/Developer/nanoclaw/
   ```

### On target

3. **Install prerequisites:**
   ```bash
   # Node.js via nvm
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
   source ~/.bashrc
   nvm install 22

   # Claude Code
   npm install -g @anthropic-ai/claude-code
   ```

4. **Clone your fork:**
   ```bash
   git clone https://github.com/YOUR-FORK/nanoclaw.git ~/Developer/nanoclaw
   ```

5. **Install dependencies and build** (native modules must be compiled for target platform):
   ```bash
   cd ~/Developer/nanoclaw
   npm install
   npm run build
   ```

6. **Build the container image** (architecture-specific — always rebuild on target even if same arch):
   ```bash
   ./container/build.sh
   ```

7. **Fix Docker socket permissions** (Linux — if user was recently added to docker group):
   ```bash
   # Immediate fix
   sudo setfacl -m u:$(whoami):rw /var/run/docker.sock

   # Persistent fix (survives Docker restarts)
   sudo mkdir -p /etc/systemd/system/docker.service.d
   sudo tee /etc/systemd/system/docker.service.d/socket-acl.conf > /dev/null <<EOF
   [Service]
   ExecStartPost=/usr/bin/setfacl -m u:$(whoami):rw /var/run/docker.sock
   EOF
   sudo systemctl daemon-reload
   ```

   > **Note:** The `[Service]` and `ExecStartPost` lines must have no leading spaces — systemd won't parse indented unit directives.

8. **Create mount allowlist:**
   ```bash
   npx tsx setup/index.ts --step mounts -- --empty
   ```

9. **Create and start the systemd service** (generates unit with correct Node.js and project paths for this machine):
   ```bash
   npx tsx setup/index.ts --step service
   systemctl --user start nanoclaw
   ```

10. **Verify:**
    ```bash
    npx tsx setup/index.ts --step verify
    tail -f logs/nanoclaw.log
    ```

11. **Channel auth:**
    - **Telegram, Slack, Discord** — token-based, transfers via `.env`. No re-authentication needed.
    - **WhatsApp** — device-specific auth (`store/auth/creds.json`). Must re-authenticate on the new machine with `/add-whatsapp`.

---

## Lessons learned

1. **Named volumes vs bind mounts** — named volumes are opaque. Always prefer bind mounts to `~/docker/` so data is portable and easy to inspect/backup. Verify what's actually mounted with `docker inspect container-name --format '{{ json .HostConfig.Binds }}'`.

2. **`ONECLI_BIND_HOST=0.0.0.0` is required on Linux** — without it, agent containers cannot reach OneCLI through the Docker bridge. Symptom: `502 Bad Gateway` or `connection refused` in container logs, `decrypt error` in OneCLI gateway logs.

3. **pg_dump is more reliable than volume copying for postgres** — tar-piping postgres data files can fail silently if postgres is running. pg_dump is clean and architecture-independent.

4. **OneCLI encryption key and database are a pair** — the `secret-encryption-key` encrypts secrets in the database. Lose either one, or let them get out of sync, and you must re-register all secrets.

5. **Don't stop the source containers until the target is verified** — keep source running as fallback until everything is confirmed end-to-end.

6. **ARM64 image compatibility** — Apple Silicon Mac and Raspberry Pi 4/5 are both ARM64, so images transfer without issues. If migrating from Intel Mac → Pi, you'd need to rebuild images on the Pi.

7. **SSH non-interactive sessions don't source `.bashrc`** — nvm and other shell tools installed via `.bashrc` won't be in PATH. Either symlink to `/usr/local/bin/` or ensure `.bash_profile` sources `.bashrc`.

8. **`node_modules` must not be copied** — native modules (like `better-sqlite3`) are compiled for a specific OS and architecture. Always run `npm install` fresh on the new machine.

9. **The systemd unit has hardcoded paths** — it contains the absolute Node.js binary path and project directory. Always regenerate it on the target with `npx tsx setup/index.ts --step service`.

10. **systemd unit files are whitespace-sensitive** — `[Service]` and directive lines must start at column 0. Leading spaces cause silent parse failures.

---

## Quick checklist

- [ ] Stop NanoClaw service on source (keep Docker running)
- [ ] Export OneCLI volumes (`onecli-app-data` via tar + pg_dump for postgres)
- [ ] Copy Docker stack data, compose files, and `.env` to target
- [ ] Copy NanoClaw repo data (`store/`, `groups/`, `data/`, `.env`) to target
- [ ] Copy `~/.onecli/` to target
- [ ] Install Node.js 22+, Docker, build tools, Claude Code on target
- [ ] Clone fork, `npm install && npm run build`
- [ ] `./container/build.sh`
- [ ] Edit `~/.onecli/docker-compose.yml` — bind mounts, `ONECLI_BIND_HOST=0.0.0.0`
- [ ] Start OneCLI, restore pg_dump if needed, verify `onecli secrets list`
- [ ] Fix Docker socket permissions (Linux: `setfacl`)
- [ ] Create mount allowlist
- [ ] Create and start systemd service
- [ ] Verify with `npx tsx setup/index.ts --step verify`
- [ ] Send test message
- [ ] Confirm working, then stop source containers
