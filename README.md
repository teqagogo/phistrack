<!--
  ██████╗ ██╗  ██╗██╗███████╗██╗  ██╗████████╗██████╗  █████╗  ██████╗██╗  ██╗
  ██╔══██╗██║  ██║██║██╔════╝██║  ██║╚══██╔══╝██╔══██╗██╔══██╗██╔════╝██║ ██╔╝
  ██████╔╝███████║██║███████╗███████║   ██║   ██████╔╝███████║██║     █████╔╝
  ██╔═══╝ ██╔══██║██║╚════██║██╔══██║   ██║   ██╔══██╗██╔══██║██║     ██╔═██╗
  ██║     ██║  ██║██║███████║██║  ██║   ██║   ██║  ██║██║  ██║╚██████╗██║  ██╗
  ╚═╝     ╚═╝  ╚═╝╚═╝╚══════╝╚═╝  ╚═╝   ╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝
-->

# PhishTrack

> *"Your credentials were leaked. So what?"*

Everybody gets phished. That's weather, not an event. The interesting question isn't
*if* your creds leak — it's what happens in the **minutes after**. Do they actually get
used? How fast? From where?

So we stopped guessing and built a machine that measures it. PhishTrack takes suspected
phishing URLs, quietly plants **credentials we own** in the fake login forms, and then sits
back and watches a farm of honeypots to see **who** turns up to use them, **when**, and
**how**. The gap between "credential planted" and "credential abused" — the **time-to-abuse**
— is the number nobody could measure before. Our record so far is *16 seconds*.

Classic honeypots wait to be found. We plant the bait ourselves. Call it the canary gambit.

**This is defensive tooling.** The honeypots serve fake data, the canaries are yours, and
you point the whole thing at infrastructure **you own and operate**. Don't be a muppet.

---

## TL;DR — get it running

```bash
git clone <this-repo> phishtrack && cd phishtrack
sudo ./install.sh                       # Docker-based, generates keys, brings the stack up
```

Dashboard lands on **http://<your-ip>:8000**. Log in with the `OPERATOR_API_KEY` the
installer just printed (also in your `.env`). Hit **Settings → Stability → Self-test**,
watch everything go green, submit a URL, done. The rest of this README is for when you want
to understand what you just unleashed — or when something catches fire.

---

## What it actually does

Four jobs, one box:

```
  COLLECT ──▶ BAIT ──▶ TRAP ──▶ MEASURE
```

- **COLLECT** — pull suspect URLs from wherever they live: manual paste, the API, IMAP /
  Graph / Exchange mailboxes, Mattermost channels, uploaded HTML, or managed git / HTTP
  threat feeds. Each URL gets opened in a stealthed headless browser, screenshotted,
  fingerprinted for known phishing kits, and scored by a calibrated ML ensemble
  (LightGBM + a char-TCN + a distilled transformer + an HTML-DOM model, fused by a
  calibrated referee that explains itself with SHAP).
- **BAIT** — when a login form is detected, we submit a **unique fake identity** ("meet
  Margaux, she isn't real") drawn from a pool of ~1.9M. Unique per URL, so a later hit
  names the exact page that sold you out. Unambiguous attribution, for free.
- **TRAP** — a farm of honeypots that look like the real thing: OWA / Citrix / VPN / portal
  HTTP lures **plus** raw-protocol listeners for SSH, RDP, VNC and MySQL. Every header,
  cookie, JA3/JA4, keystroke and payload is captured, stored instantly, and enriched
  asynchronously (GeoIP, ASN, Tor, AbuseIPDB, VirusTotal).
- **MEASURE** — correlate leak → reuse per canary, compute time-to-abuse, cluster campaigns,
  and export it all as STIX / MISP / CSV IOCs. Watch it happen live on an attack globe.

It never phones home. The analyzer can run air-gapped; GeoIP and the ML seed model ship
offline in the box.

---

## Architecture

```
        the internet's worst                     YOUR turf
        ───────────────────                     ───────────
   phishing page                          ┌────────────────────────────┐
        │  (1) URL in                      │  phishtrack-main  :8000     │
        ▼                                  │  ├─ dashboard + /api/*       │
   ┌─────────────┐   (2) render+score      │  ├─ analyzer worker loop     │
   │  ANALYZER   │─────────────────────────▶│  ├─ ML ensemble + SHAP      │
   │ (playwright │                          │  ├─ canary vault            │
   │  + stealth) │◀── (3) plant canary ─────│  └─ SQLite core (WAL)       │
   └─────────────┘                          └───────────┬────────────────┘
                                                         │ report-only
   attacker  ──(4) tries stolen creds──▶  HONEYPOTS       │ (INTERNAL_API_KEY)
                                          :8443 citrix     │
                                          :8444 owa        ▼
                                          :8445 vpn   enrich async → alerts,
                                          :8446 portal      globe, IOC export
                                          :2222 ssh
                                          :3389 rdp
                                          :5900 vnc
                                          :3306 mysql
```

Two trust planes, two keys: the human dashboard talks `OPERATOR_API_KEY`; the honeypots talk
`INTERNAL_API_KEY` and are report-only — a popped honeypot can't read your loot or tap the
`/ws` stream. More on that below, because getting it wrong is a great way to lock yourself
out.

---

## The stack

- **Python 3.11+**, FastAPI + Uvicorn, all state in a single **SQLite** DB (WAL mode).
- **Playwright** driving system **Chromium** for URL rendering, with `playwright-stealth` and
  `curl_cffi` JA3/JA4 TLS impersonation for the "fetch like a victim" path.
- **ML**: `lightgbm` + `onnxruntime` + `numpy`. Ships a seed model; the trainer is optional.
- **YARA** (`yara-python`) over captured payloads.
- **Docker / Podman + Compose** for the recommended deployment; bare-metal + systemd is fully
  supported too.

Full pinned list in [`requirements.txt`](requirements.txt). The ML / YARA / mail bits all
degrade gracefully — if a dependency or model is missing, that feature switches off instead
of taking the process down with it.

---

## Installation

You've got four ways in. Pick your fighter.

### 1. Docker, the easy way (recommended)

```bash
sudo ./install.sh
```

`install.sh` is idempotent and upgrade-aware. It will:

1. Detect your distro/arch and install system deps (Docker/Podman, Python, Chromium libs) —
   works across apt / dnf / pacman / apk / zypper.
2. Generate a `.env` with **two fresh random 64-hex keys** (never reuses a default — there
   is no default).
3. Download the offline GeoIP DBs into `geoip/` at build time.
4. Build the images and bring up the **production** profile.
5. Wait for `:8000/health` and `:8080/health` to go green, then seed the legit-domain
   allowlists (Tranco top-1M + public auth domains).
6. If it finds an existing install, it takes an online SQLite `.backup`, snapshots your data
   dirs, and runs additive schema migrations — your data survives the upgrade.

When it's done it prints your dashboard URL and the operator key. That key is the only thing
between the internet and your loot, so treat it accordingly.

### 2. Docker Compose by hand

Prefer to drive it yourself? Fine.

```bash
cp .env.example .env
# generate the two keys:
python3 -c "import secrets; print('INTERNAL_API_KEY=' + secrets.token_hex(32))" >> .env
python3 -c "import secrets; print('OPERATOR_API_KEY=' + secrets.token_hex(32))" >> .env

docker compose --profile production up -d --build
```

**Mind the profile.** `--profile production` brings up `phishtrack-main` plus the eight
per-service honeypots. The `test` profile's combined `phishtrack-honeypot` *also* binds
`:8443` and will fight the Citrix honeypot for it — don't mix profiles unless you enjoy port
clashes (see Troubleshooting).

### 3. Bare-metal + systemd (no Docker)

`install.sh` also does a native install: it creates a service user, drops a Python
virtualenv at **`.venv/`**, installs the pinned deps, wires Playwright to your system
Chromium, and writes two systemd units (`phishtrack.service` on `:8000`, honeypot on
`:8080`). Same script, it figures out which path to take based on whether Docker is present.

Manual equivalent, if you want to see the moving parts:

```bash
python3 -m venv .venv
.venv/bin/pip install -U pip wheel setuptools
.venv/bin/pip install -r requirements.txt
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 .venv/bin/playwright install-deps   # use system chromium
.venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000 --workers 1
```

### 4. Raspberry Pi

```bash
sudo ./install_pi.sh
```

Same idea, ARM-tuned, ships the full package set and installs Chromium for you.

---

## Configuration

Everything lives in `.env` (see [`.env.example`](.env.example) for the annotated template).
The only **required** entries are the two keys; everything else has sane defaults baked in.

| Var | What it's for |
|-----|---------------|
| `INTERNAL_API_KEY` | honeypots ↔ main app, machine-to-machine. **Required.** |
| `OPERATOR_API_KEY` | the human dashboard + `/api/*` login key. Falls back to `INTERNAL_API_KEY` if unset (single-key mode — fine for a lab, not for prod). |
| `PHISHTRACK_ANALYZER_WORKERS` | analyzer fan-out. Auto-clamped to `min(requested, cpu, RAM÷1.5GB)` so you can't OOM yourself. Leave empty to auto-tune. |
| `PHISHTRACK_HIT_BATCH_N` / `_MS` / `_QUEUE_MAX` | honeypot hit-ingestion batching under flood. Defaults handle a 5k-req burst fine. |
| `PHISHTRACK_GEOIP_ENABLED` | `1` (default) uses the bundled offline DB-IP Lite mmdb; `0` disables geo enrichment. |
| `PHISHTRACK_UPGRADE_RECLAIM_DAYS` / `_SKIP_RAW_RECLAIM` / `_RETENTION_DELETE_CHUNK` | data-retention knobs. Safe defaults; the one-time reclaim only runs on upgrade. |
| `PHISHTRACK_ML_PUBLISH_GATE` / `_MIN_AUC` | ML champion/challenger publish gate. Only relevant if you run the trainer. |

Analyzer throughput, retention windows, ML verdict mode (shadow → assist → primary) and the
notification channels are all **live-tunable from Settings** — no restart needed.

### The two-key thing, because you'll get it wrong once

- Dashboard won't accept your key? You're probably pasting `INTERNAL_API_KEY`. The dashboard
  wants `OPERATOR_API_KEY`.
- **First run is gated on literal peer trust**: the very first auth must come from a genuinely
  local/trusted connection. A spoofed `X-Forwarded-For` cannot unlock the key — by design.
  If you're behind a reverse proxy, set your trusted proxies up first (see below) or do the
  first login from the box itself.
- Programmatic clients authenticate with the **`X-API-Key` header**, *not* a `?key=` query
  param. The self-test and every `/api/*` call included.

### Behind a reverse proxy (X-Forwarded-For)

By default PhishTrack trusts the raw TCP peer only — it will **not** believe an `XFF` header
from just anyone, because attackers send those too. If you front it with nginx/Traefik/
Cloudflare, tell it which proxies to trust so client-IP attribution stays honest. Check what
it currently thinks with `GET /api/system/client-ip`.

---

## First contact — is it alive?

```bash
# liveness (no auth)
curl -s localhost:8000/health          # dashboard — always up
curl -s localhost:8080/health          # honeypot health (bare-metal / test profile)

# full subsystem self-test (auth required — note the HEADER, not ?key=)
curl -s -X POST -H "X-API-Key: $OPERATOR_API_KEY" localhost:8000/api/selftest | jq

# one-shot operational snapshot
./status.sh            # ports, pids, cpu/mem, queue depth, honeypot health
./status.sh json       # same, machine-readable
```

Green self-test = all subsystems (DB, analyzer, honeypots, ML, enrichment) are wired
correctly. If a check is red, the message tells you which one; cross-reference Troubleshooting.

---

## Driving it day-to-day

- **Submit a URL**: paste it in the dashboard, or
  `POST /api/analyze {"url": "..."}` with your operator key. Bulk import up to 50k at a time
  via `POST /api/analyze/bulk`, or drop an HTML file at `POST /api/analyze/html`.
- **Feeds**: wire up managed git or HTTP threat feeds, or one-click the built-in presets
  (Daily-Mal-Phishing, OpenPhish) from Settings → Feeds. Auto-discovers URLs / domains / IPs.
- **Mail & chat**: connect IMAP / Graph / Exchange mailboxes and Mattermost channels to
  ingest reported phish automatically. Credentials are Fernet-encrypted at rest.
- **Watch the traps**: the Honeypot Monitor, the attack Globe (great-circle arcs from
  attacker → honeypot, click for the session dossier), and the Time-to-Abuse tab with the
  Top-10 fastest-abuse leaderboard.
- **Grab the loot**: the "Pot" tab dedupes captured credentials, exploits and IOCs; export
  as STIX / MISP / CSV / encrypted ZIP.
- **Domain blacklist**: a drop-list applied at ingest so you never waste an analyzer slot on
  noise you've already decided about.

There's a bundled Python CLI example for every one of these under `static/examples/`, kept
in lock-step with the API.

---

## Operations

```bash
./start.sh          # bring the stack up
./status.sh         # health + metrics (add `json` for machine output)
./reset.sh          # wipe data back to a clean slate (asks first)
./reset.sh --db-only    # keep screenshots/payloads, reset just the DB
./reset.sh --force      # skip the confirmation prompt (you monster)
./uninstall.sh      # tear it all down
sudo ./install.sh   # re-run to upgrade in place (backs up + migrates first)
```

**Backups**: the DB is a single SQLite file. `install.sh` snapshots it on upgrade, but for
routine backups use the online backup so you don't copy a half-written WAL:

```bash
sqlite3 data/phishing_tracker.db ".backup 'backup-$(date +%F).db'"
```

---

## Troubleshooting — when it all goes sideways

The greatest-hits collection. Symptom → why → fix.

### `database is locked`

The classic. SQLite with WAL mode under a multi-worker/multi-process load: if the WAL never
checkpoints it grows without bound, contention exceeds `busy_timeout`, and every write reads
as "locked". We run a background `PRAGMA wal_checkpoint(TRUNCATE)` worker exactly to stop
this, but a giant mailbox import + heavy analyzer fan-out can still outrun it.

**Fix**, in order:
```bash
# 1. is the WAL enormous?
ls -lh data/phishing_tracker.db-wal
# 2. stop the writers (compose down, or stop the systemd units)
# 3. force a truncating checkpoint
sqlite3 data/phishing_tracker.db "PRAGMA wal_checkpoint(TRUNCATE); VACUUM;"
# 4. bring it back up
```
To make it checkpoint more aggressively, drop `wal_checkpoint_sec` in Settings (default 60s,
clamped 15s–1h). If you're importing a monster mailbox, throttle the analyzer workers first.

### `Address already in use` / a port won't bind

PhishTrack wants a fistful of ports: **8000** (dashboard), **8080** (honeypot mux, test
profile), **8443** citrix, **8444** owa, **8445** vpn, **8446** portal, **2222** ssh,
**3389** rdp, **5900/5901** vnc, **3306** mysql.

- **`:3306` clash** → you've got a real MySQL/MariaDB already listening. Stop it, or remap the
  honeypot's published port in `docker-compose.yml`.
- **`:8443` clash on startup** → you almost certainly mixed the `test` and `production`
  profiles. The test profile's `phishtrack-honeypot` grabs `:8443` too. Use **only**
  `--profile production`. `docker compose down` and come back up clean.
- Find the culprit: `sudo ss -tlnp | grep -E ':(8000|8080|8443|3306)'`.

### Analyzer isn't draining the queue / URLs stuck "pending"

Check `./status.sh` — it shows `queue_pending` / `queue_active`. If active is 0 while pending
climbs, the analyzer loop isn't running or is starved.

- Confirm Chromium launches (next entry).
- Remember the fan-out is **RAM-gated**: `PHISHTRACK_ANALYZER_WORKERS` is clamped to
  `RAM÷1.5GB`. On a 2GB box you get one worker no matter what you ask for. Give it more RAM
  or accept the throughput.

### Playwright / Chromium won't launch

Symptoms: analyzer errors about the browser, or every URL comes back `unreachable`.

```bash
# in Docker this is baked in; bare-metal, install the browser deps against SYSTEM chromium:
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 .venv/bin/playwright install-deps
which chromium chromium-browser        # make sure one exists
```
We deliberately use system Chromium (`PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1`) so we're not
shipping a 300MB browser per image. If your distro hides it somewhere weird, the analyzer's
launch path checks the usual names.

### Self-test / API returns `401 Unauthorized`

You used `?key=` instead of the header. Every `/api/*` call wants
**`-H "X-API-Key: <key>"`**. And use the **operator** key for dashboard/loot endpoints, the
**internal** key only for honeypot-report endpoints.

### Can't log into the dashboard at all

- Wrong key (operator vs internal — see above).
- First-run auth gate: the first login must come from a trusted peer. Behind a proxy without
  trusted-proxy config, a spoofed `XFF` won't let you in (that's the point). Do the first
  login from the host, or configure trusted proxies, then check `GET /api/system/client-ip`.

### A honeypot shows unhealthy

```bash
docker compose --profile production ps        # which service is down?
docker compose --profile production logs honeypot-citrix --tail=50
```
Usual causes: a port clash (above), or the container hit its memory/pids cap under a flood
(the raw-protocol honeypots run `read_only`, `cap_drop: ALL`, with mem/cpu/pids limits — a
brute-force burst can trip a limit rather than let the box fall over). Bump the limit in
`docker-compose.yml` if you're genuinely getting hammered.

### ML predictions look wrong / stuck, or a model won't update

The models live in the `ml_models` Docker volume, seeded from the shipped bundle on first
boot. Gotcha: the **volume shadows the image** — once seeded it won't pick up a newer bundle
just because you rebuilt the image. If you've retrained or shipped a new seed and it's not
taking:
```bash
docker compose --profile production down
docker volume rm <project>_ml_models      # forces a re-seed from the current bundle
docker compose --profile production up -d
```
Also: ML runs **shadow** by default (computed, never touches the verdict). Promote it to
assist/primary in Settings → Analysis when you trust it. If `lightgbm`/`onnxruntime` failed
to import, detection quietly falls back to the heuristic scorer — check the startup logs.

### Trainer dies with `EACCES` on `data/train_feeds`

Only if you run the optional trainer. The trainer runs as uid 1000 inside its container; if
the host `data/train_feeds` is root-owned it can't write. `install.sh` chowns it to the
container uid for you, but if you created the dir by hand:
```bash
sudo chown -R 1000:1000 data/train_feeds
```

### GeoIP enrichment is blank

The offline `geoip/*.mmdb` files aren't tracked in git (they're ~140MB) — they're downloaded
at image build. If you're running bare-metal or copied the tree without them, geo comes back
empty. Either drop the DB-IP Lite mmdb files into `geoip/`, or set
`PHISHTRACK_GEOIP_ENABLED=0` to stop it trying.

### `docker compose build --no-cache` fills the disk

Building all the honeypots from scratch pulls a lot of Chromium/Playwright layers. If your
disk gasps:
```bash
docker builder prune -af && docker image prune -af    # keeps named volumes (ml_models etc.)
```
Then build `phishtrack-main` first and let the honeypots build sequentially.

### YARA rules didn't load

`yara-python` is optional; if it's missing, kit-YARA over captured payloads just switches off
(everything else keeps working). The community rule set is fetched at runtime and bounded in
size — a failed fetch is non-fatal. Install `yara-python` if you want it back.

### Nuke it from orbit

When you just want a clean slate:
```bash
./reset.sh              # interactive, wipes all data
./reset.sh --db-only    # keep the captured artefacts, reset the DB
```

---

## Port reference

| Port | Service | Profile |
|------|---------|---------|
| 8000 | Dashboard + `/api/*` | always |
| 8080 | Honeypot mux | test |
| 8443 | Citrix / NetScaler lure | production |
| 8444 | OWA / Exchange lure | production |
| 8445 | VPN / GlobalProtect lure | production |
| 8446 | Generic portal lure | production |
| 2222 | SSH honeypot | production |
| 3389 | RDP honeypot | production |
| 5900 / 5901 | VNC honeypot | production |
| 3306 | MySQL honeypot | production |

---

## Repo map

| Path | What lives there |
|------|------------------|
| `main.py` | FastAPI app: dashboard, all `/api/*` routes, the analyzer worker loop. Yes, it's one big file. It's load-bearing now — tread carefully. |
| `analyzer.py`, `evasion.py` | URL analysis pipeline + anti-bot/WAF/CAPTCHA evasion |
| `*_honeypot.py`, `honeypot.py`, `deception.py`, `*_responder.py` | the honeypot services, their decoys and exploit responders |
| `enrichment.py` | GeoIP/ASN/reputation enrichment + the 26 exploit signatures |
| `canary_identities.py` | the fake-identity generator (~1.9M-address space) |
| `ml/`, `train/` | inference ensemble + the (optional) training pipeline |
| `database.py` | SQLite schema, queries, retention, migrations. Start here if it says "locked". |
| `templates/`, `static/` | the single-page dashboard (+ `static/examples/` API clients) |
| `pycrawlbypass/` | self-contained, public-technique-only evasion helpers |
| `install.sh`, `install_pi.sh`, `start.sh`, `status.sh`, `reset.sh`, `uninstall.sh` | the ops toolkit |

See [`DOCUMENTATION.html`](DOCUMENTATION.html) for the full feature tour and
[`RELEASE_NOTES.md`](RELEASE_NOTES.md) for the feature set and version history.

---

## A word on intent

This exists to **measure and attribute credential abuse against infrastructure you own and
operate**. The honeypots serve fake data; the canaries are yours; the point is to learn how
fast the bad guys move so you can move faster. Point it at your own deployment, plant your own
bait, and don't be the reason we can't have nice things.

## License

Licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE) for the full text.
