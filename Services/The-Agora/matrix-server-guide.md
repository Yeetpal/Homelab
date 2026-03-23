# Self-Hosted Matrix Server — Setup & Troubleshooting Guide
 
> A practical reference built from real-world deployment experience using **Synapse**, **PostgreSQL**, **Dockhand**, **Cloudflare Tunnel**, and **mautrix-discord**. All commands are production-tested.
 
---
 
> ### 🔑 Placeholder Key
> Wherever you see text in `ALL_CAPS_WITH_UNDERSCORES`, replace it with your own value before running the command.
>
> | Placeholder | What to enter |
> |---|---|
> | `YOUR_MATRIX_DOMAIN` | Your Matrix subdomain e.g. `matrix.yourdomain.com` |
> | `YOUR_BASE_DOMAIN` | Your root domain e.g. `yourdomain.com` |
> | `YOUR_USERNAME` | Your Matrix username e.g. `admin` |
> | `YOUR_SERVER_LOCAL_IP` | Local IP of your Ubuntu server e.g. `192.168.0.x` |
> | `YOUR_DOCKHAND_STACK_PATH` | Full path to your stack's data volume — find it by running `sudo docker inspect matrix-synapse --format='{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'` |
> | `YOUR_DISCORD_GUILD_ID` | Your Discord server ID (right-click server → Copy Server ID) |
> | `YOUR_DISCORD_CHANNEL_ID` | A specific Discord channel ID (right-click channel → Copy Channel ID) |
> | `YOUR_POSTGRES_PASSWORD` | The password you set for the Postgres `synapse` user |
> | `YOUR_TUNNEL_TOKEN` | Token from Cloudflare Zero Trust → Tunnels |
| `YOUR_TURN_TOKEN_ID` | Turn Token ID from Cloudflare Realtime → TURN Service |
| `YOUR_TURN_API_TOKEN` | API Token from Cloudflare Realtime → TURN Service |
 
---
 
## Table of Contents
 
1. [Phase 1 — Initial Server Setup](#phase-1--initial-server-setup)
2. [Phase 2 — Docker Stack via Dockhand](#phase-2--docker-stack-via-dockhand)
3. [Phase 3 — Common Startup Crashes](#phase-3--common-startup-crashes)
4. [Phase 4 — Admin Account Creation](#phase-4--admin-account-creation)
5. [Phase 5 — Cloudflare Tunnel Setup](#phase-5--cloudflare-tunnel-setup)
6. [Phase 6 — Federation with Cloudflare Worker](#phase-6--federation-with-cloudflare-worker)
7. [Phase 7 — Video Calls (Cloudflare TURN)](#phase-7--video-calls-cloudflare-turn)
8. [Phase 8 — Synapse Admin Dashboard](#phase-8--synapse-admin-dashboard)
9. [Phase 9 — Element Call (RTC Focus)](#phase-9--element-call-rtc-focus)
10. [Phase 10 — Discord Bridge (mautrix-discord)](#phase-10--discord-bridge-mautrix-discord)
11. [Useful Commands Reference](#useful-commands-reference)
 
---
 
## Phase 1 — Initial Server Setup
 
### 1.1 Create data directories
 
```bash
sudo mkdir -p /opt/matrix/synapse
sudo mkdir -p /opt/matrix/db
```
 
### 1.2 Generate Synapse config
 
> ⚠️ Replace `YOUR_MATRIX_DOMAIN` before running.
>
> 🔒 **Critical:** The `server_name` value you set here defines your Matrix IDs (e.g. `@user:YOUR_MATRIX_DOMAIN`) and **cannot be changed** after users or rooms are created. Choose your domain carefully before proceeding.
 
```bash
sudo docker run -it --rm \
    -v /opt/matrix/synapse:/data \
    -e SYNAPSE_SERVER_NAME=YOUR_MATRIX_DOMAIN \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```
 
### 1.3 Switch database to PostgreSQL
 
Open the config:
```bash
sudo nano /opt/matrix/synapse/homeserver.yaml
```
 
Find the `database:` section. Delete the SQLite block and replace with:
 
```yaml
database:
  name: psycopg2
  allow_unsafe_locale: true   # Required if Postgres was initialised without C locale
  args:
    user: synapse
    password: YOUR_POSTGRES_PASSWORD
    database: synapse
    host: db
    cp_min: 5
    cp_max: 10
```
 
> ⚠️ `allow_unsafe_locale` must sit at the `database:` level, **not** inside `args:` — placing it inside `args:` will cause a crash.
 
### 1.4 Fix file permissions
 
```bash
sudo chown -R 991:991 /opt/matrix/synapse
```
 
---
 
## Phase 2 — Docker Stack via Dockhand
 
Paste this into a new Dockhand stack (e.g. `matrix-stack`).
 
> ⚠️ Replace `YOUR_POSTGRES_PASSWORD` and `YOUR_MATRIX_DOMAIN`.
 
```yaml
version: '3.8'
 
services:
  db:
    image: postgres:15
    container_name: matrix-db
    environment:
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: YOUR_POSTGRES_PASSWORD
      POSTGRES_DB: synapse
      POSTGRES_INITDB_ARGS: --encoding=UTF8 --lc-collate=C --lc-ctype=C
    volumes:
      - /opt/matrix/db:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse"]
      interval: 5s
      timeout: 5s
      retries: 5
 
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: matrix-synapse
    ports:
      - "8008:8008"
    volumes:
      - /opt/matrix/synapse:/data
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
```
 
> The `healthcheck` on `db` prevents Synapse attempting to connect before Postgres is ready — the most common cause of crash loops.
 
---
 
## Phase 3 — Common Startup Crashes
 
### 3.1 Crash: `IncorrectDatabaseSetup` — wrong collation
 
**Symptom:**
```
IncorrectDatabaseSetup: Database has incorrect collation of 'en_US.utf8'. Should be 'C'
```
 
**Fix A — Correct collation at init time (preferred for fresh installs):**
 
```bash
sudo rm -rf /opt/matrix/db/*
```
 
Then add to the `db` service environment in your compose file:
```yaml
POSTGRES_INITDB_ARGS: --encoding=UTF8 --lc-collate=C --lc-ctype=C
```
Redeploy. Postgres will reinitialise with the correct collation.
 
**Fix B — Allow unsafe locale (if data already exists):**
 
In `homeserver.yaml`, place `allow_unsafe_locale` at the correct level:
 
```yaml
database:
  name: psycopg2
  allow_unsafe_locale: true   # ← must be HERE, NOT inside args:
  args:
    user: synapse
    ...
```
 
### 3.2 Crash: `invalid connection option "allow_unsafe_locale"`
 
You placed `allow_unsafe_locale` inside `args:`. Move it up one level — see the correct structure in §3.1 Fix B.
 
### 3.3 Crash: Database race condition
 
**Symptom:** `psycopg2.OperationalError: connection to server at "db" failed`
 
Postgres takes ~15s to initialise on first boot. Add `healthcheck` + `condition: service_healthy` to your compose (already included in Phase 2 above).
 
### 3.4 OOM Kill / constant restarts with no clear error
 
Check kernel logs:
```bash
dmesg -T | grep -i oom
```
 
Matrix Synapse + Postgres requires a **minimum of 2 GB RAM**. Resize your server if under this threshold.
 
---
 
## Phase 4 — Admin Account Creation
 
### 4.1 Create via Dockhand terminal (or SSH)
 
```bash
sudo docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008
```
 
Follow prompts:
- **New user localpart:** `YOUR_USERNAME`
- **Make admin:** `yes`
 
### 4.2 Promote existing user to admin
 
```bash
sudo docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  -u YOUR_USERNAME -a --exists-ok \
  http://localhost:8008
```
 
### 4.3 Verify server is running
 
Open in a browser:
```
http://YOUR_SERVER_LOCAL_IP:8008/_matrix/client/versions
```
A JSON response confirms Synapse is live.
 
---
 
## Phase 5 — Cloudflare Tunnel Setup
 
### 5.1 Create the tunnel token
 
1. Cloudflare Dashboard → **Zero Trust → Networks → Tunnels**
2. Create tunnel → choose Docker → copy the **Tunnel Token**
 
### 5.2 Add cloudflared to your Dockhand stack
 
```yaml
  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare-tunnel
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=YOUR_TUNNEL_TOKEN
```
 
### 5.3 Configure public hostname in Cloudflare
 
In your tunnel's **Public Hostname** tab, add:
 
| Field | Value |
|---|---|
| Subdomain | `matrix` |
| Domain | `YOUR_BASE_DOMAIN` |
| Service Type | `HTTP` |
| URL | `YOUR_SERVER_LOCAL_IP:8008` |
 
### 5.4 Tell Synapse to trust the proxy
 
In `homeserver.yaml`:
 
```yaml
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false
```
 
---
 
## Phase 6 — Federation with Cloudflare Worker
 
Allows other Matrix servers to find yours on port 443 instead of the default 8448.
 
### 6.1 Create a Cloudflare Worker
 
1. Cloudflare Dashboard → **Workers & Pages → Create Worker**
2. Replace the default code with:
 
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const headers = {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    };
 
    // Server-to-server federation
    if (url.pathname === "/.well-known/matrix/server") {
      return new Response(
        JSON.stringify({ "m.server": "YOUR_MATRIX_DOMAIN:443" }),
        { headers }
      );
    }
 
    // Client discovery + Element Call support
    if (url.pathname === "/.well-known/matrix/client") {
      return new Response(
        JSON.stringify({
          "m.homeserver": {
            "base_url": "https://YOUR_MATRIX_DOMAIN"
          },
          "org.matrix.msc4143.rtc_foci": [
            {
              "type": "livekit",
              "livekit_service_url": "https://jwt.call.element.io"
            }
          ]
        }),
        { headers }
      );
    }
 
    return new Response("Not Found", { status: 404 });
  }
};
```
 
> ℹ️ The `rtc_foci` block is included here from the start. If you deploy this Worker without it and add it later, you **must** purge the Cloudflare cache for `.well-known/matrix/client` — otherwise Cloudflare will keep serving the old response. See §9.1 for the purge steps.
 
### 6.2 Add a route
 
In the Worker's **Settings → Domains & Routes → Add Route**:
 
| Field | Value |
|---|---|
| Route | `YOUR_MATRIX_DOMAIN/.well-known/matrix/*` |
| Zone | `YOUR_BASE_DOMAIN` |
 
> Use your **Matrix subdomain** here (not root domain) if your `server_name` in `homeserver.yaml` is `YOUR_MATRIX_DOMAIN`.
 
### 6.3 Test federation
 
Visit [federationtester.matrix.org](https://federationtester.matrix.org) and enter `YOUR_MATRIX_DOMAIN`.
 
---
 
## Phase 7 — Video Calls (Cloudflare TURN)
 
> Uses Cloudflare Realtime TURN so no ports need to be opened on your router.
 
### 7.1 Get TURN credentials
 
Cloudflare Dashboard → **Realtime → TURN Service → Create TURN App**
 
Copy the **Turn Token ID** and **API Token**.
 
### 7.2 Add matrix-turnify to your Dockhand stack
 
```yaml
  matrix-turnify:
    image: ghcr.io/bpbradley/matrix-turnify:latest
    container_name: matrix-turnify
    restart: unless-stopped
    environment:
      - CLOUDFLARE_CALLS_APP_ID=YOUR_TURN_TOKEN_ID
      - CLOUDFLARE_CALLS_APP_TOKEN=YOUR_TURN_API_TOKEN
      - SYNAPSE_URL=http://matrix-synapse:8008
```
 
### 7.3 Add TURN settings to homeserver.yaml
 
```yaml
turn_uris:
  - "turn:turn.cloudflare.com:3478?transport=udp"
  - "turn:turn.cloudflare.com:3478?transport=tcp"
  - "turns:turn.cloudflare.com:5349?transport=tcp"
turn_shared_secret: "cloudflare-calls-placeholder"
turn_user_lifetime: 86400000
turn_allow_guests: true
```
 
### 7.4 Add Cloudflare Tunnel route for TURN
 
In your existing tunnel's **Public Hostname** tab → Add route:
 
| Field | Value |
|---|---|
| Subdomain | `matrix` |
| Domain | `YOUR_BASE_DOMAIN` |
| Path | `/_matrix/client/v3/voip/turnServer` |
| Service Type | `HTTP` |
| URL | `YOUR_SERVER_LOCAL_IP:4499` |
 
> Note: Cloudflare will concatenate the path, so it should appear as `YOUR_MATRIX_DOMAIN/_matrix/client/v3/voip/turnServer` in the routes list.
 
---
 
## Phase 8 — Synapse Admin Dashboard
 
### 8.1 Add synapse-admin to your stack
 
```yaml
  synapse-admin:
    image: awesometechnologies/synapse-admin:latest
    container_name: synapse-admin
    restart: unless-stopped
    ports:
      - "8080:80"
```
 
Access at `http://YOUR_SERVER_LOCAL_IP:8080`
 
### 8.2 Login details
 
| Field | Value |
|---|---|
| Homeserver URL | `http://YOUR_SERVER_LOCAL_IP:8008` |
| Username | `@YOUR_USERNAME:YOUR_MATRIX_DOMAIN` |
| Password | As set during registration |
 
> ⚠️ Do not use `localhost` — the dashboard runs in **your browser**, so localhost points to your PC, not the server.
 
### 8.3 CORS fix (if "Something went wrong")
 
In `homeserver.yaml`, add at the very bottom (not indented under any other section):
 
```yaml
cors_origins:
  - "*"
```
 
> Lock this down to a specific domain once you have HTTPS configured.
 
---
 
## Phase 9 — Element Call (RTC Focus)
 
Fixes the `MISSING_MATRIX_RTC_FOCUS` error in newer Element clients.
 
> ℹ️ This phase can be completed any time after Phase 6 — it does not depend on Phase 7 or 8. If you are seeing the `MISSING_MATRIX_RTC_FOCUS` error, come here directly.
 
---
 
### ⚠️ Critical: Diagnose who is serving `.well-known` first
 
If you are using a **Cloudflare Worker** for federation (Phase 6), that Worker — not Synapse — is serving the `.well-known/matrix/client` response. Editing `homeserver.yaml` will have **no effect** on what the outside world sees.
 
Run these two commands to confirm:
 
```bash
# Test what the public internet sees (via Cloudflare)
curl https://YOUR_MATRIX_DOMAIN/.well-known/matrix/client
 
# Test what Synapse itself would serve (bypasses Cloudflare)
curl http://localhost:8008/.well-known/matrix/client
```
 
**Interpreting results:**
 
| Public result | Localhost result | What it means |
|---|---|---|
| Old JSON (no `rtc_foci`) | `M_NOT_FOUND` | ✅ Cloudflare Worker is serving it — fix the Worker (§9.1) |
| Old JSON (no `rtc_foci`) | Old JSON (no `rtc_foci`) | Synapse is serving it — fix `homeserver.yaml` (§9.2) |
| New JSON (has `rtc_foci`) | Anything | ✅ Server is correct — clear Element cache (§9.3) |
 
---
 
### 9.1 Fix via Cloudflare Worker (most common with this setup)
 
Go to **Cloudflare Dashboard → Workers & Pages** → open your federation Worker → **Edit Code**.
 
Replace the entire script with this updated version that includes the `rtc_foci` block:
 
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const headers = {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    };
 
    // Server-to-server federation
    if (url.pathname === "/.well-known/matrix/server") {
      return new Response(
        JSON.stringify({ "m.server": "YOUR_MATRIX_DOMAIN:443" }),
        { headers }
      );
    }
 
    // Client discovery + Element Call support
    if (url.pathname === "/.well-known/matrix/client") {
      return new Response(
        JSON.stringify({
          "m.homeserver": {
            "base_url": "https://YOUR_MATRIX_DOMAIN"
          },
          "org.matrix.msc4143.rtc_foci": [
            {
              "type": "livekit",
              "livekit_service_url": "https://jwt.call.element.io"
            }
          ]
        }),
        { headers }
      );
    }
 
    return new Response("Not Found", { status: 404 });
  }
};
```
 
Click **Save and Deploy**.
 
Then purge Cloudflare's cache so the change propagates immediately:
 
1. Cloudflare Dashboard → **Caching → Configuration → Custom Purge**
2. Enter: `https://YOUR_MATRIX_DOMAIN/.well-known/matrix/client`
3. Click **Purge**
 
Verify the fix:
```bash
curl https://YOUR_MATRIX_DOMAIN/.well-known/matrix/client
```
You should now see `livekit_service_url` in the response.
 
---
 
### 9.2 Fix via homeserver.yaml (only if Synapse is serving `.well-known` directly)
 
> Only use this path if `curl http://localhost:8008/.well-known/matrix/client` returns JSON (not `M_NOT_FOUND`).
 
```bash
sudo nano /opt/matrix/synapse/homeserver.yaml
```
 
Add to the **very bottom** of the file, with no leading spaces on the first line:
 
```yaml
serve_well_known: true
 
extra_well_known_client_content:
  "org.matrix.msc4143.rtc_foci":
    - "type": "livekit"
      "livekit_service_url": "https://jwt.call.element.io"
```
 
> ⚠️ YAML is whitespace-sensitive — use spaces only, never tabs. `extra_well_known_client_content:` must be flush with the left margin.
 
Apply the change:
```bash
sudo docker stop matrix-synapse
sudo docker start matrix-synapse
```
 
Verify:
```bash
curl http://localhost:8008/.well-known/matrix/client
```
 
---
 
### 9.3 Clear the Element app cache
 
Element caches `.well-known` aggressively. Even after fixing the server, the app will continue showing the error until its cache is cleared.
 
**Desktop (Windows / macOS / Linux)**
 
The in-app "Clear Cache" button is often not enough. Do a hard reset:
 
1. Log out of Element
2. Close the app completely (check system tray)
3. Delete the data folder:
   - **Windows:** `%appdata%\Element`
   - **macOS:** `~/Library/Application Support/Element`
   - **Linux:** `~/.config/Element`
4. Relaunch and log back in
 
**Android**
 
Settings → Apps → Element → Storage → **Clear Cache** AND **Clear Data** (this logs you out)
 
**iOS**
 
There is no cache-clear button. The only reliable fix is to **uninstall and reinstall** Element from the App Store.
 
**Browser (Element Web)**
 
1. Log out
2. Press `Ctrl+Shift+Delete` (Windows) or `Cmd+Shift+Delete` (Mac)
3. Clear **Cookies and other site data** + **Cached images and files**
4. Restart browser and log back in
 
---
 
## Phase 10 — Discord Bridge (mautrix-discord)
 
> ⚠️ **Before you start — read this:** Message backfill inserts messages into Synapse one at a time. 10,000 messages may take 20–60 minutes to appear. 100,000+ may take days. The bridged room will appear completely empty until Synapse finishes processing the entire batch, then all messages appear at once. This is expected behaviour — do not restart the bridge or assume it is broken.
 
### 10.1 Prepare your Discord bot
 
1. [Discord Developer Portal](https://discord.com/developers/applications) → **New Application**
2. **Bot tab** → Enable **Server Members Intent** and **Message Content Intent**
3. **OAuth2 → URL Generator** → Scope: `bot`
4. Minimum bot permissions required:
   - View Channels
   - Send Messages
   - Read Message History
   - Add Reactions
   - Create Public Threads / Send Messages in Threads
5. Copy the generated URL, open it in a browser, and invite the bot to your Discord server
 
### 10.2 Add the bridge to your Dockhand stack
 
```yaml
  mautrix-discord:
    image: dock.mau.dev/mautrix/discord:latest
    container_name: mautrix-discord
    restart: unless-stopped
    volumes:
      - YOUR_DOCKHAND_STACK_PATH/discord-data:/data
```
 
Deploy once — the container will start, generate a blank `config.yaml`, then crash (expected).
 
### 10.3 Edit config.yaml
 
The config file will be at `YOUR_DOCKHAND_STACK_PATH/discord-data/config.yaml`.
 
If Dockhand's file manager is read-only, use the container terminal:
 
```bash
# Add sleep so container stays alive for editing
# Add to compose: command: sleep 3600
# Then:
apk add nano
nano /data/config.yaml
```
 
**Required changes:**
 
```yaml
# Under homeserver:
domain: YOUR_MATRIX_DOMAIN
address: http://matrix-synapse:8008
 
# Under appservice:
address: http://mautrix-discord:29334
 
# Under database:
database:
  type: sqlite3-fk-wal
  uri: sqlite:////data/mautrix-discord.db
 
# Under bridge → permissions:
bridge:
  permissions:
    "YOUR_MATRIX_DOMAIN": "admin"
    "@YOUR_USERNAME:YOUR_MATRIX_DOMAIN": "admin"
 
# Under bridge → backfill (optional — for message history):
  backfill:
    enabled: true
    forward_limits:
      initial:
        channel: 10000   # Do NOT use -1 — it will crash
        dm: 10000        # Do NOT use -1 — it will crash
```
 
Remove `command: sleep 3600` from compose once edits are done.
 
### 10.4 Generate registration.yaml
 
This file allows Synapse to trust the bridge. The container generates it on successful startup. If it doesn't appear:
 
```bash
# Stop container first, then:
sudo docker exec -it mautrix-discord \
  /usr/bin/mautrix-discord -g \
  -c /data/config.yaml \
  -r /data/registration.yaml
```
 
### 10.5 Link registration.yaml to Synapse
 
Add to the `synapse` service in your compose:
 
```yaml
volumes:
  - /opt/matrix/synapse:/data
  - YOUR_DOCKHAND_STACK_PATH/discord-data:/discord-data:ro
```
 
Add to the bottom of `homeserver.yaml`:
 
```yaml
app_service_config_files:
  - "/discord-data/registration.yaml"
```
 
Restart both containers.
 
### 10.6 Connect your Discord account
 
In Element, start a DM with `@discordbot:YOUR_MATRIX_DOMAIN` and send:
 
```
login
```
 
Follow the prompts. Then bridge your server:
 
```
guilds status
guilds bridging-mode YOUR_DISCORD_GUILD_ID everything
guilds bridge YOUR_DISCORD_GUILD_ID --entire
```
 
---
 
## Troubleshooting: Discord Bridge Backfill Issues
 
### Problem: Room is empty after backfill
 
**Cause A — Component Type 20 crash (Discord Checkpoint / Year in Review cards)**
 
These interactive cards crash older bridge versions during backfill.
 
Fix: Find and delete any "Discord Checkpoint" / "Year in Review" cards from the Discord channel, then reset and retry.
 
**Cause B — `-1` limit causes a panic crash**
 
```
panic: runtime error: slice bounds out of range [:-1]
```
 
Fix: Set all limits in `config.yaml` to a positive integer (e.g. `10000`). Never use `-1`.
 
**Cause C — Bridge thinks it already finished ("newer than metadata")**
 
```
DBG Not backfilling, last message in database is newer than last message in metadata
```
 
Fix: Perform a surgical database reset (see below).
 
---
 
### Surgical database reset for a specific channel
 
> Use this when a channel room is stuck or empty and the bridge won't retry.
 
```bash
# 1. Stop the bridge
sudo docker stop mautrix-discord
 
# 2. Wipe portal and message records for the broken channel
sudo docker run --rm \
  -v YOUR_DOCKHAND_STACK_PATH/discord-data:/data \
  alpine sh -c "
    apk add --quiet --no-cache sqlite &&
    sqlite3 /data/mautrix-discord.db \
      \"DELETE FROM portal WHERE dcid = 'YOUR_DISCORD_CHANNEL_ID';
       DELETE FROM message WHERE dc_chan_id = 'YOUR_DISCORD_CHANNEL_ID';\"
  "
 
# 3. Leave the Matrix room in Element
 
# 4. Start the bridge
sudo docker start mautrix-discord
```
 
> The bridge will auto-invite you to a fresh room and trigger the initial backfill.
 
---
 
### Full bridge factory reset
 
> Use when multiple rooms are broken or the bridge state is corrupted.
 
```bash
# In Element — send to bridge bot DM:
# guilds unbridge YOUR_DISCORD_GUILD_ID
# delete-all-portals
# logout
 
# Then in terminal:
sudo docker stop mautrix-discord
 
sudo docker run --rm \
  -v YOUR_DOCKHAND_STACK_PATH/discord-data:/data \
  alpine sh -c "
    apk add --quiet --no-cache sqlite &&
    sqlite3 /data/mautrix-discord.db \
      \"DELETE FROM portal; DELETE FROM message;\"
  "
 
sudo docker start mautrix-discord
 
# In Element — reconnect:
# login
# guilds bridge YOUR_DISCORD_GUILD_ID --entire
```
 
---
 
### Reload a specific channel with a higher message limit
 
```bash
# 1. Update config.yaml:
#    Set channel: and dm: limits to desired number (e.g. 50000)
 
# 2. Stop bridge and wipe channel memory
sudo docker stop mautrix-discord
 
sudo docker run --rm \
  -v YOUR_DOCKHAND_STACK_PATH/discord-data:/data \
  alpine sh -c "
    apk add --quiet --no-cache sqlite &&
    sqlite3 /data/mautrix-discord.db \
      \"DELETE FROM portal WHERE dcid = 'YOUR_DISCORD_CHANNEL_ID';
       DELETE FROM message WHERE dc_chan_id = 'YOUR_DISCORD_CHANNEL_ID';\"
  "
 
# 3. Leave the room in Element
 
# 4. Start bridge — auto-invite will trigger new backfill
sudo docker start mautrix-discord
```
 
> ⚠️ Messages insert **one by one** on standard Synapse. 10,000 messages may take 20–60 minutes to appear. 100,000+ may take days. The room will appear empty until Synapse finishes processing the entire batch, then messages appear all at once.
 
---
 
## Useful Commands Reference
 
```bash
# Check all running containers
sudo docker ps
 
# View live logs for a container
sudo docker logs -f matrix-synapse --tail 100
sudo docker logs -f mautrix-discord --tail 100
 
# Restart a container
sudo docker restart matrix-synapse
sudo docker restart mautrix-discord
 
# Find homeserver.yaml on disk
sudo docker inspect matrix-synapse \
  --format='{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
 
# Create a new Matrix user (registration must be enabled, or use admin flag)
sudo docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008
 
# Promote existing user to admin
sudo docker exec -it matrix-synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  -u YOUR_USERNAME -a --exists-ok http://localhost:8008
 
# Inspect SQLite database schema (bridge)
sudo docker run --rm \
  -v YOUR_DOCKHAND_STACK_PATH/discord-data:/data \
  alpine sh -c "
    apk add --quiet --no-cache sqlite &&
    sqlite3 /data/mautrix-discord.db '.schema message'
  "
 
# Force-generate bridge registration file
sudo docker exec -it mautrix-discord \
  /usr/bin/mautrix-discord -g \
  -c /data/config.yaml \
  -r /data/registration.yaml
```
 
---
 
## Notes
 
- **server_name** in `homeserver.yaml` defines your Matrix ID (e.g. `@user:YOUR_MATRIX_DOMAIN`). This **cannot be changed** after users are created.
- Always stop the bridge container **before** running SQLite surgery — if the container is running it will rewrite its RAM cache back to disk and undo the wipe.
- The bridge sends messages to Synapse one at a time (no bulk insert on standard Synapse). Large backfills will show an empty room until Synapse finishes processing the entire batch.
- Forwarded messages and interactive UI components (Discord "Checkpoint" cards) in a channel's history can crash older bridge versions during backfill. Delete them from Discord before retrying.
