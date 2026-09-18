# Fleet Telemetry System — Full Documentation

A full-stack drone fleet telemetry prototype: a **C++ simulator** flies 10
autonomous delivery drones around **IIT Mandi North Campus**, snapshots the
fleet to a **JSON file** every 2 seconds, a **FastAPI shim** serves that file,
and a **React + Leaflet dashboard** renders it live on a map.

```text
sim/sim.cpp (C++, 10 drones, 2 s tick)
  → backend/telemetry.json (file snapshot, overwritten every tick)
    → api/main.py (FastAPI, GET /telemetry on :8000)
      → dashboard/display/src/App.tsx (React 19 + Leaflet, polls every 2 s on :5173)
```

There is **no database, no websocket, and no `POST /telemetry` ingestion**.
The JSON file is the shared bus between the simulator and the API.

---

## 1. Repository layout

```text
Fleet-Telemetry-System/
├── Makefile                     # builds the simulator binary
├── simulator                    # built binary (gitignored, produced by `make`)
├── common/
│   ├── types.h                  # shared structs + state enum
│   ├── id.h / id.cpp            # drone UID generator
├── data/
│   └── locations.h              # base station + destination pool (world model)
├── sim/
│   ├── drone.h                  # Drone entity class (data + setters only)
│   └── sim.cpp                  # physics loop + state machine + main()
├── backend/
│   ├── backend.h                # C++ snapshot store + JSON serializer
│   ├── telemetry.json           # runtime snapshot (gitignored, rewritten every 2 s)
│   └── README.md                # schema note for telemetry.json
├── api/
│   └── main.py                  # FastAPI shim: reads telemetry.json, serves GET /telemetry
├── dashboard/display/
│   ├── package.json             # Vite + React 19 + react-leaflet + leaflet
│   └── src/
│       ├── App.tsx              # entire dashboard UI + map + polling
│       ├── App.css              # dark sidebar + topbar styling
│       ├── main.tsx             # React entry point
│       └── index.css            # global styles
├── docs/
│   └── README.md                # this file
├── EXPLANATION.md               # deep deconstruction (state machine, battery model, physics)
└── INSTRUCTIONS.md              # startup instructions (Linux)
```

---

## 2. File-by-file reference

### 2.1 `Makefile` — build definition

| Line | Content |
|------|---------|
| 2–3 | `CXX = g++`, `CXXFLAGS = -std=c++17 -Wall -I.` (the `-I.` lets includes like `"common/types.h"` resolve from the repo root) |
| 6 | `SRC = sim/sim.cpp common/id.cpp` — the only two translation units |
| 9 | `TARGET = simulator` — output binary name (no `.exe` extension; on Ubuntu, Files opens `.exe` files with Archive Manager, so the binary was renamed from `sim.exe`) |
| 12–13 | `all:` compiles everything with one `g++` invocation |
| 16–17 | `clean:` removes `simulator` (plus legacy `sim.exe` if present) |

Run from the repo root: `make` → `./simulator`.

---

### 2.2 `common/types.h` — shared vocabulary (30 lines)

The contract every layer implicitly agrees on. Included by
`sim/drone.h`, `backend/backend.h`, and `data/locations.h`.

| Line | Content |
|------|---------|
| 6–16 | `enum STATES { OFF, START, TAKEOFF, CRUISE, APPROACH, DELIVERY, RETURNING, LANDED, CHARGING }` — 9 states. Note: `START` and `APPROACH` are never assigned by the simulator; only 7 states are live |
| 18–22 | `struct Position { lat, lng, alt }` — a drone's live point in space |
| 24–28 | `struct Location { lat, lng, addr }` — a named place (base or destination). Field is `addr` in C++; the JSON serializer writes it as `"address"`, which is what the dashboard reads |

---

### 2.3 `common/id.h` / `common/id.cpp` — UID generator (6 lines each)

| File | Role |
|------|------|
| `id.h:4` | Declares `int generateUID();` |
| `id.cpp:3-6` | Defines it: `static int staticId = 100; return ++staticId;` |

Every `Drone` construction calls `generateUID()`, so a fresh run with
`NUM_OF_DRONES = 10` yields IDs **101–110** in construction order.
Caveats: not thread-safe, not persistent across restarts (a second run
reuses the same IDs), and the counter keeps incrementing within one
process (a respawned drone keeps its original ID — IDs are assigned at
construction, never recycled).

---

### 2.4 `data/locations.h` — the world model (68 lines)

Defines *where* the simulation happens: IIT Mandi North Campus,
Kamand Valley (31.7812939, 76.9975020).

| Line | Content |
|------|---------|
| 10–19 | `STREET_NAMES` — 24 campus/village road names (`North Campus Main Rd`, `Kamand Valley Rd`, `Salgi Village Rd`, …) used to fabricate destination addresses |
| 21–39 | `generateDestinations(count)` — deterministic generator (`mt19937` seeded with `42`, so the pool is identical every run): uniform `lat ∈ [31.765, 31.795]`, `lng ∈ [76.985, 77.010]` (~3 km box around campus), house number `∈ [100, 9999]`, address suffix `", Kamand Valley, HP"` |
| 41–43 | `BASES` — a **single** entry: `{31.7812939, 76.9975020, "BASE STATION"}` (North Campus Main Gate). All drones spawn, return to, and recharge at this one station |
| 45 | `DESTINATIONS = generateDestinations(1000)` — the 1000-point destination pool the simulator draws from |
| 50–68 | Commented-out test fixtures (old Las Vegas bases/destinations) — dead reference, kept for testing ideas only |

To change the operating area, edit the `latDist`/`lngDist` ranges and/or
`STREET_NAMES` here and rebuild — no other file hardcodes geography
(the dashboard map center in `App.tsx` should be updated to match).

---

### 2.5 `sim/drone.h` — the Drone entity (75 lines)

Pure data holder + setters. **No behaviour, no physics** — the state
machine lives in `sim.cpp`. Fields (`:9-15`): `id`, `pos`, `battery`,
`state`, `destination`, `base`, `speed`.

| Line | Content |
|------|---------|
| 17–21 | Default constructor: ID from `generateUID()`, position origin, battery 100, state `OFF` |
| 22–29 | Main constructor `(base, destination, latOffset, lngOffset)`: spawns near the base (`base + offset`, alt 0), battery 100, state `OFF`, default speed `0.0002` deg/tick (immediately overwritten by the simulator's speed distribution) |
| 31–37 | Getters: `getId / getPosition / getBattery / getState / getDestination / getBase / getSpeed` |
| 39–48 | `setSpeed / setBase / setDestination` |
| 50–54 | `movePos(dLat, dLng, dAlt)` — blindly adds deltas; all navigation math is the caller's job |
| 56–59 | `drainBattery(amount)` — subtracts, floored at 0 |
| 63–67 | `setPosition(lat, lng, alt)` — absolute teleport (used on respawn after charging) |
| 69–72 | `chargeBattery(amount)` — adds, capped at 100 |

---

### 2.6 `sim/sim.cpp` — physics loop + state machine (196 lines)

The core of the system. One process, one thread, infinite 2-second tick.

**Configuration & RNG (`:13-33`)**

| Line | Content |
|------|---------|
| 13–14 | RNG: `random_device`-seeded `mt19937` (unlike destinations, drone assignment is **non-deterministic** per run) |
| 17–19 | Distributions: `destDist` over `DESTINATIONS`, spawn `offsetDist ±0.0003°` (~±30 m), `speedDist 0.00015–0.00035` deg/tick (~15–35 m/tick) |
| 25 | `NUM_OF_DRONES = 10` |
| 26–33 | `CRUISE_ALTITUDE = 30.0`, `DELIVERY_WAIT_TICKS = 3`, `CHARGE_PER_TICK = 4`, `LOW_BATTERY_THRESHOLD = 20`, drain every 5th tick (`-2` moving / `-1` idle) |

**Helpers (`:35-49`)** — `distance2D()` (defined but unused — CRUISE/RETURNING inline their own `sqrt`) and `maybeDrainBattery()` (drains only when `tick % 5 == 0`).

**Initialization (`:51-84`)** — creates `droneRegistry` (ground truth `vector<Drone>`), `fleet` (`DroneList` serializable mirror), and two parallel countdown arrays. Each drone gets the single `BASES.front()`, a random destination, a spawn offset, and a random speed; all 10 registrations are printed to stdout.

**State machine (`:88-188`)** — every tick, every drone:

| State | Behaviour |
|-------|-----------|
| `OFF` | Transient → `TAKEOFF` |
| `TAKEOFF` | Climb `+5` alt/tick until `alt >= 30` → `CRUISE` |
| `CRUISE` | Battery `<= 20` → `RETURNING`. Else fly one `speed` step toward destination (normalized lat/lng vector); arrival within `0.0001°` (~11 m) → `DELIVERY` with 3-tick countdown |
| `DELIVERY` | Wait 3 ticks → `RETURNING` |
| `RETURNING` | Fly toward base; on arrival descend `-5`/tick, then → `LANDED` |
| `LANDED` | Transient → `CHARGING` |
| `CHARGING` | `+4` battery/tick until 100, then respawn at base + fresh offset, pick a **new** destination (`:175`), → `TAKEOFF`. Cycle repeats forever — no terminal state |

**Output (`:187-194`)** — each drone is pushed into `fleet` via
`fleet.update(DroneState(d))`, then the whole fleet is rewritten to the
CWD-relative path `backend/telemetry.json`, `Tick N written` is printed,
and the loop sleeps 2000 ms. **Must be launched from the repo root**,
otherwise the write fails with `Failed to write telemetry`.

---

### 2.7 `backend/backend.h` — C++ snapshot store + serializer (156 lines)

Despite the directory name, this is **not a server** — it is a
header-only C++ library used by the simulator. No HTTP, no Python here.

| Line | Content |
|------|---------|
| 14–27 | `stateToString()` — maps the `STATES` enum to `"OFF" … "CHARGING"` strings for JSON |
| 29–74 | `class DroneState` — immutable-ish snapshot of one drone (`id, pos, battery, state, destination, base, speed`) plus `last_updated` (serialization time, identical for all drones within a tick — not physics time). Constructed from a `Drone` (`:41-49`) |
| 76–104 | `class DroneList` — `unordered_map<int, DroneState>` with `addDrone` / `update` / `getDroneState` (all O(1)) and `size()` |
| 105–153 | `writeTelemetry(filename)` — O(N) full-file rewrite every tick via manual `ofstream` string building. Note the field rename: C++ `addr` → JSON `"address"` (`:135,140`). No string escaping, no atomic write (tmp-file + rename) — an API read landing mid-write can see torn JSON |
| 133–142 | Per-drone JSON shape: `id, position{lat,lng,alt}, battery, state, timestamp, base{lat,lng,address}, destination{lat,lng,address}, speed` |

---

### 2.8 `backend/telemetry.json` — the shared bus (gitignored)

The runtime snapshot file. Overwritten **in full, every 2 s** by the
simulator (`sim.cpp:190`); read on every dashboard poll by the API.
Schema: `{ "drones": [ {id, position, battery, state, timestamp, base, destination, speed}, … ] }` (10 entries). See also `backend/README.md` for the minimal schema note. Never hand-edit — it is regenerated continuously while the sim runs. Timestamps update every tick.

---

### 2.9 `api/main.py` — FastAPI shim (48 lines)

A thin file-read bridge between `telemetry.json` and the dashboard.
Needs only `fastapi` + `uvicorn`. **Must be started from inside `api/`**
because the file path is CWD-relative.

| Line | Content |
|------|---------|
| 5–12 | App + CORS allowing only `http://localhost:5173` (the Vite default) |
| 14–16 | `GET /` → `{"message": "Home route, hello!"}` (health check) |
| 18–21 | `GET /telemetry` → `json.load(open("../backend/telemetry.json"))` — full file parse per request, no validation, no caching, no POST endpoint |
| 26–48 | Commented-out stale test fixture (uses `IDLE`/`DELIVERING` states that don't exist in the sim) |

Run: `cd api && uvicorn main:app --port 8000`. Verify: `curl http://127.0.0.1:8000/telemetry | head -c 500`.

---

### 2.10 `dashboard/display/` — React + Leaflet frontend

**`package.json`** — `react 19`, `react-leaflet 5`, `leaflet 1.9.4`, built/served by `vite 8` + `typescript 5.9`. Scripts: `dev` (port 5173), `build` (`tsc -b && vite build`), `preview`, `lint`.

**`src/main.tsx`** — React entry point; mounts `App`.

**`src/App.tsx`** (324 lines) — the entire dashboard:

| Line | Content |
|------|---------|
| 15–33 | TS types: `Location{lat,lng,address}`, `Drone{id, position{lat,lng,alt}, battery, state, base, destination, timestamp}` — mirrors the JSON shape from `backend.h` |
| 35 | `LOW_BATTERY_THRESHOLD = 25` — note the mismatch: the sim aborts at 20, the UI flags at 25, so the dashboard raises `LOW_BATTERY` before the sim turns the drone around |
| 39–57 | `stateColor()` — battery ≤ 25 → red; CRUISE/TAKEOFF/APPROACH/DELIVERY → blue; RETURNING → amber; CHARGING/LANDED/OFF → grey. Collapses 7 sim states into 4 visual buckets |
| 59–69 | `normalizeState()` — same thresholds → `LOW_BATTERY / ACTIVE / RETURNING / IDLE` filter buckets |
| 71–83 | `FocusDrone` — `flyTo(drone, zoom 15, 0.8 s)` when a drone is selected |
| 92–115 | Polling: `fetch("http://127.0.0.1:8000/telemetry")` on mount + `setInterval(2000)` (frequency-matched to the sim tick) |
| 117–134 | `metrics` via `useMemo` O(N): `total / active / returning / lowBattery / idle / avgBattery` |
| 141–210 | Sidebar: brand, critical-alerts card, filter buttons (ALL/ACTIVE/RETURNING/LOW_BATTERY/IDLE), selected-drone panel (status, battery, altitude, destination) |
| 213–240 | Topbar: total / active / avg battery / alerts + critical banner when `lowBattery > 0` |
| 247–254 | Map header + `MapContainer` centered `[31.7812939, 76.997502]` (BASE STATION) at `zoom 15` with OSM tiles |
| 262–318 | Per-drone `CircleMarker` (tooltip on hover, popup on click with id/battery/state/base/destination); selected drone gets a dashed `Polyline base → position → destination` route |

**`src/App.css` / `src/index.css`** — dark 320 px sidebar + light topbar, alert banner, map shell, loading state. No logic.

---

### 2.11 `EXPLANATION.md` / `INSTRUCTIONS.md` — companion docs

| File | Role |
|------|------|
| `EXPLANATION.md` | Olympiad-style deconstruction: state-machine proof, battery-model math, map-physics critique (degree-space vs haversine), known quirks (threshold mismatch, flight at 0% battery, torn reads, stale artifacts) |
| `INSTRUCTIONS.md` | Linux startup runbook: prerequisites, 3-terminal bring-up (simulator → API → dashboard), ports/URLs table, troubleshooting matrix |

---

## 3. Data flow (what actually happens at runtime)

```text
1. ./simulator (repo root)                     2 s tick
   sim.cpp state machine advances 10 drones
     → fleet.update(...) per drone (O(1))
     → fleet.writeTelemetry("backend/telemetry.json") (O(N) full rewrite)
2. uvicorn main:app (from api/)                per dashboard poll
   GET /telemetry → json.load("../backend/telemetry.json")
3. npm run dev (from dashboard/display)        every 2 s
   App.tsx fetches :8000/telemetry → setDrones → Leaflet re-renders
```

Effective rate: **1 snapshot / 2 s** end to end (not per-drone streaming).

---

## 4. Run order (summary)

Terminal 1 (repo root): `make && ./simulator` — leave running.
Terminal 2 (`api/`): `pip install fastapi uvicorn && uvicorn main:app --port 8000`.
Terminal 3 (`dashboard/display/`): `npm install && npm run dev` → open `http://localhost:5173`.

Details, port conflicts, and failure modes: see `INSTRUCTIONS.md`.
Internals, proofs, and load-bearing flaws: see `EXPLANATION.md`.
