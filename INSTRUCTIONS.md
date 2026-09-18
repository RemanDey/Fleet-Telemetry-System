# Fleet Telemetry System — Startup Instructions (Linux only)

Full-stack drone fleet telemetry prototype:

```text
sim/sim.cpp (C++, 100 drones, 2s tick)
  → backend/telemetry.json (file snapshot, overwritten every tick)
    → api/main.py (FastAPI, GET /telemetry on :8000)
      → dashboard/display (React 19 + Leaflet + Vite on :5173, polls every 2s)
```

There is no database, no websocket, and no `POST /telemetry` ingestion
(despite what `docs/README.md` claims). The JSON file is the shared bus.

You need **3 terminals running at the same time, started in order**:
1. Simulator, 2. API, 3. Dashboard.

---

## 0. Prerequisites

Tested on Ubuntu/Debian Linux.

| Tool | Needed for | Check | Install (Debian/Ubuntu) |
|------|------------|-------|--------------------------|
| `g++` (C++17) | simulator | `g++ --version` | `sudo apt update && sudo apt install -y g++ make` |
| `make` | simulator build | `make --version` | same as above |
| `python3` + `pip` | FastAPI API | `python3 --version && pip3 --version` | `sudo apt install -y python3 python3-pip python3-venv` |
| `node` + `npm` | dashboard | `node -v && npm -v` | install Node 20+ from [nodejs.org](https://nodejs.org/) or via `nvm`, then `npm -v` |

Clone / enter the repo:

```bash
cd Fleet-Telemetry-System
pwd   # must show .../Fleet-Telemetry-System for Step 1
ls    # should show: api/ backend/ common/ dashboard/ data/ sim/ Makefile
```

---

## 1. Quick Start (TL;DR)

Terminal 1 (repo root):

```bash
make
./simulator
```

Terminal 2 (API):

```bash
cd api
pip install fastapi uvicorn
uvicorn main:app --port 8000
```

Terminal 3 (dashboard):

```bash
cd dashboard/display
npm install
npm run dev
```

Open:

- API check: `http://127.0.0.1:8000/telemetry`
- Dashboard: `http://localhost:5173`

Details and troubleshooting below — read them if anything fails.

---

## 2. Step 1 — C++ Simulator (Terminal 1)

**Working directory matters.** The simulator writes to the relative path
`backend/telemetry.json`. You **must** run it from the repo root.
Do not double-click the binary in Files — run it from a terminal.

```bash
# from repo root: .../Fleet-Telemetry-System
make
ls -l simulator backend/telemetry.json
./simulator
```

What to expect:

- Builds with `g++ -std=c++17 -Wall -I. sim/sim.cpp common/id.cpp -o simulator` (see `Makefile`).
- Runs forever in a `while(true)` loop, one tick every 2000 ms.
- Overwrites `backend/telemetry.json` every tick with 100 drones.
- Prints per-tick console output. Leave it running. `Ctrl+C` to stop.

Verify in another shell:

```bash
ls -l backend/telemetry.json
# timestamp should update every ~2s while simulator runs
watch -n 1 ls -l backend/telemetry.json
```

Rebuild from scratch:

```bash
make clean
make
```

> If you see `Failed to write telemetry`, you started `simulator` from the
> wrong directory. `cd` back to the repo root and rerun `./simulator`.

---

## 3. Step 2 — FastAPI Backend (Terminal 2)

`api/main.py` is a thin file-read shim:

- `GET /` → `{"message": "Home route, hello!"}`
- `GET /telemetry` → `json.load(open("../backend/telemetry.json"))`

That `../backend/telemetry.json` path is **relative to your shell CWD**,
so you **must** start uvicorn from inside `api/`.

### 3a. Install Python deps with pip

```bash
cd api              # .../Fleet-Telemetry-System/api
pwd                 # confirm you are in api/

# Option A — system pip (simplest):
pip3 install fastapi uvicorn

# Option B — venv (recommended on Ubuntu 23.04+ where PEP 668
# blocks system pip; use this if `pip install` errors with
# "externally-managed-environment"):
python3 -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn
```

You only need `fastapi` + `uvicorn`. There is no `requirements.txt` in this repo.

### 3b. Run the API

```bash
# still inside .../Fleet-Telemetry-System/api, venv activated if you use one
uvicorn main:app --port 8000
```

Expected log:

```text
INFO:     Started server process
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

Verify:

```bash
curl http://127.0.0.1:8000/
# {"message":"Home route, hello!"}

curl http://127.0.0.1:8000/telemetry | head -c 500
# {"drones": [{"id": 101, ...}, ...]}
```

Keep this terminal running.

> Do NOT run `uvicorn api.main:app` from the repo root — the
> `../backend/telemetry.json` open will resolve to the wrong path and
> return `FileNotFoundError`. Always `cd api` first.

---

## 4. Step 3 — React Dashboard (Terminal 3)

Stack (see `dashboard/display/package.json`): Vite + React 19 +
`react-leaflet@5` + `leaflet@1.9.4`.

The app hardcodes `fetch("http://127.0.0.1:8000/telemetry")` in
`dashboard/display/src/App.tsx:92` and polls every 2s. The API's CORS
rule (`api/main.py`) only allows `http://localhost:5173`, which is the
Vite default — so use that URL.

### 4a. Install Node deps with npm

```bash
cd dashboard/display   # .../Fleet-Telemetry-System/dashboard/display
pwd                    # confirm

npm install
```

This reads `package.json` / `package-lock.json` and installs into `node_modules/`.
Re-run it after any `git pull` that changes `package.json`.

### 4b. Run dev server

```bash
# still inside dashboard/display
npm run dev
```

Expected output:

```text
VITE ... ready in ... ms
➜  Local:   http://localhost:5173/
```

Open `http://localhost:5173` in a browser. You should see:

- Leaflet map centered on Las Vegas `[36.1699, -115.1398]`
- 100 `CircleMarker` drones, color-coded by battery/state
- Sidebar metrics (total / active / returning / low battery / avg battery)
- Click a drone for `Popup` with id, battery, state, base, destination + dashed route polyline

Keep this terminal running. `Ctrl+C` to stop.

Optional production build:

```bash
npm run build
npm run preview   # serves dist/ locally
```

---

## 5. Ports & URLs Summary

| Service | Dir to run from | Command | URL |
|---------|----------------|---------|-----|
| Simulator | repo root | `./simulator` | writes `backend/telemetry.json` (no port) |
| API | `api/` | `uvicorn main:app --port 8000` | `http://127.0.0.1:8000/telemetry` |
| Dashboard dev | `dashboard/display/` | `npm run dev` | `http://localhost:5173` |

If a port is busy:

```bash
ss -tlnp | grep -E '8000|5173'
# kill the PID shown, or use another port:
uvicorn main:app --port 8001   # note: dashboard still fetches :8000, so prefer killing :8000
npm run dev -- --port 5174     # note: API CORS only allows :5173, so prefer killing :5173
```

---

## 6. Stop / Clean

Press `Ctrl+C` in each of the 3 terminals, in reverse order (dashboard → API → sim).

```bash
# repo root
make clean      # removes simulator
rm -rf api/.venv dashboard/display/node_modules  # full clean (optional)
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Failed to write telemetry` | sim started outside repo root | `cd` to repo root, rerun `./simulator` |
| `FileNotFoundError: ../backend/telemetry.json` from uvicorn | uvicorn started outside `api/` | `cd api`, rerun `uvicorn main:app --port 8000` |
| `curl :8000/telemetry` returns stale speeds (~0.001 vs code's 0.00015–0.00035) | checked-in `telemetry.json` is old | start the sim; it overwrites the file every 2s |
| Dashboard empty map / `Failed to fetch` / CORS error | API not running | start Step 2 first, confirm `curl :8000/telemetry` works, use `http://localhost:5173` exactly |
| `pip install` → `externally-managed-environment` (Ubuntu) | PEP 668 | use `python3 -m venv .venv && source .venv/bin/activate` then `pip install fastapi uvicorn` |
| `npm run dev` → `vite: not found` | skipped `npm install` | `cd dashboard/display && npm install` |
| Torn JSON / occasional API 500 while sim writes | sim does non-atomic `ofstream` rewrite each tick, API reads mid-write | retry the request; for a real fix use tmp-file + rename in `backend/backend.h` |
| Dashboard shows `LOW_BATTERY` but sim still flies drone | threshold mismatch: frontend 25 (`App.tsx`) vs sim 20 (`sim.cpp`) | expected demo quirk; unify thresholds to fix |
| Drones fly at `battery: 0` | sim has no dead-stick model | expected demo quirk |

---

## 8. Known Quirks (do not "fix" by changing CWD)

- All paths are CWD-relative by design: sim → `backend/telemetry.json`, API → `../backend/telemetry.json`. Changing launch dirs breaks them.
- Data rate is 1 snapshot / 2s, not "50 updates/s" as `docs/README.md` claims.
- `EXPLANATION.md` has a full deconstruction (state machine, battery model, map physics) if you want internals.
