# Fleet-Telemetry-System: Olympiad-Calibre Deconstruction

## 0. One-Line Theorem

This is a **3-stage, 4-language, file-coupled cyber-physical simulation**: `C++` generates physics → `JSON file` acts as shared memory → `Python/FastAPI` serves it → `React/Leaflet` renders it. There is no database, no websocket, no real backend ingestion.

```text
sim/sim.cpp (C++, N=100 agents, 0.5Hz loop)
  → backend/telemetry.json (overwritten snapshot, O(N) each tick)
    → api/main.py (FastAPI, file read on GET /telemetry)
      → dashboard/display/src/App.tsx (poll 2s, O(N) render on Leaflet)
```

Shared contracts: `common/types.h`, `data/locations.h`.

---

## 1. Axioms: `common/`

### `common/types.h:6-16` — `enum STATES`

```cpp
OFF, START, TAKEOFF, CRUISE, APPROACH, DELIVERY, RETURNING, LANDED, CHARGING
```

**Lemma 1 (Dead states):** `START, APPROACH` are never assigned in `sim/sim.cpp:94-185`. They are dead code. Only 7 states are live.

### `common/types.h:18-28` — Geometry

```cpp
struct Position { lat,lng,alt; };
struct Location { lat,lng,addr; };
```

Note the naming fracture that propagates: C++ uses `addr`, JSON/C++ serializer uses `"address"` (`backend/backend.h:135,140`), frontend type uses `address` (`dashboard/display/src/App.tsx:18`). This is correct only by manual mapping, no schema enforcement.

### `common/id.cpp:3-6` — UID Generator

```cpp
static int staticId = 100; return ++staticId;
```

**Invariant:** Single run with `NUM_OF_DRONES=100` yields IDs `101..200` exactly — confirmed by `backend/telemetry.json` (100 objects, ~22 lines each, 2204 lines total).

**Failure modes (contest pitfalls):**

1. Not thread-safe, not persistent — restart collides IDs.
2. Copy semantics: `Drone d(...); droneRegistry.push_back(d); fleet.addDrone(d)` — ID is copied, not re-generated. Correct, but subtle.

---

## 2. World Model: `data/locations.h`

* `BASES`: single fixed hub `BASE STATION` at IIT Mandi North Campus Main Gate (31.7812939, 76.9975020).
* `DESTINATIONS`: `generateDestinations(1000)` with `mt19937(42)` — **deterministic**. Same 1000 addresses every run. `lat~U[36.05,36.28]`, `lng~U[-115.35,-115.05]`, `house~U[100,9999] + 24 street names`.
* Seeded RNG = reproducible test fixture. Good olympiad practice.

---

## 3. Agent: `sim/drone.h` — `class Drone`

Pure data + setters, no behavior. Fields: `id, pos, battery(0-100 clamped), state, destination, base, speed`.

Default `speed=0.0002` deg/tick, immediately overwritten by `speedDist~U[0.00015,0.00035]` in `sim/sim.cpp:19,72`.

> **Anomaly detected:** `backend/telemetry.json` shows `speed ~0.0008-0.0017`, ~5x larger than code's distribution. Proof the checked-in JSON is **stale**, from an older binary. Never trust artifacts over source.

Methods are `O(1)`: `movePos`, `drainBattery` (floor 0), `chargeBattery` (cap 100), all with clamping invariants.

---

## 4. Physics Engine: `sim/sim.cpp` — The Core

### 4.1 Initialization `sim/sim.cpp:64-80`

For each `i < 100`: pick `base~Uniform(BASES)`, `dest~Uniform(DESTINATIONS)`, `offset~U[-0.0003,0.0003]^2`, spawn near base. Push to two parallel structures:

* `vector<Drone> droneRegistry` — ground truth.
* `DroneList fleet` — serializable mirror.
* `deliveryTicksRemaining[100]`, `chargingTicksRemaining[100]` — parallel arrays (fragile, index-coupled).

### 4.2 Kinematics — The Only Math

```cpp
distance = sqrt(dLat*dLat + dLng*dLng)              // sim/sim.cpp:35-39,114,145
pos += step * (target-pos)/distance                 // sim/sim.cpp:122-125,157-160
```

This is **normalized gradient descent in lat/lng degree space** with fixed step `speed` deg/tick. Termination radius `0.0001 deg ~= 11m`.

**Olympiad critique:**

1. Ignores Earth curvature. At 36N, 1 deg lng ~= 90km vs 1 deg lat ~=111km — ~20% anisotropy distorted into isotropic steps. Acceptable for 30km Vegas grid (<1% absolute error), wrong for real navigation. Correct fix: haversine + meters.
2. `distance2D()` is defined but never called — CRUISE/RETURNING inline their own `sqrt`. Dead helper.
3. Vertical and horizontal decoupled: `TAKEOFF: alt+=5` until `CRUISE_ALTITUDE=30`, `RETURNING`: horizontal first, then `alt-=5`. Cruise at `30` labelled `ft` in `App.tsx:199` — 30ft cruise is physically absurd (real: 200-400ft). Unit ambiguity.

### 4.3 Battery Automaton

```cpp
maybeDrainBattery(tick, isMoving): if tick%5==0: -2 else -1
CHARGE_PER_TICK=+4/tick, LOW_BATTERY_THRESHOLD=20
```

Effective rates: `0.4%/tick` moving, `0.2%/tick` idle, `+4%/tick` charging. Asymmetric by 10x — deliberate to guarantee liveness (charge faster than drain).

**Critical invariant violation:** Frontend `LOW_BATTERY_THRESHOLD=25` (`App.tsx:35`) != Simulator `20`. Dashboard raises `LOW_BATTERY` before sim aborts. By design or bug, the two layers disagree on "critical".

**Liveness bug visible in data:** many drones in `RETURNING` with `battery=0` — proof drones fly at 0% (no dead-stick model). Safety property `battery>0 ==> flight` is violated.

### 4.4 State Machine — Formal Proof

| State | Guard → Action |
|---|---|
| `OFF` | unconditional → `TAKEOFF` (1 tick transient) |
| `TAKEOFF` | `alt<30 → alt+=5`, else → `CRUISE` |
| `CRUISE` | `batt<=20 → RETURNING`; `dist<1e-4 → DELIVERY(3 ticks)`; else step toward `dest` |
| `DELIVERY` | countdown 3 → `RETURNING` |
| `RETURNING` | step toward `base`; if arrived: descend, then → `LANDED` |
| `LANDED` | unconditional → `CHARGING` (1 tick transient) |
| `CHARGING` | `batt<100 → +4`, else respawn at `base+offset`, new `dest`, → `TAKEOFF` |

**Theorem (Liveness):** All paths cycle `...→CHARGING→TAKEOFF→...` infinitely. No absorbing state. Verified: no `break`, `while(true)` at `sim/sim.cpp:88`, sleep 2000ms at `:194`.

### 4.5 Output `sim/sim.cpp:187-191`

```cpp
fleet.update(DroneState(d)); // O(1) hashmap replace, refreshes timestamp
fleet.writeTelemetry("backend/telemetry.json"); // O(N) full rewrite EVERY tick
```

Path is CWD-relative — must launch from repo root, else silent `Failed to write telemetry`. No atomic rename → **torn-read race** with FastAPI reader.

Build: `Makefile: CXX=g++ -std=c++17 -Wall -I. SRC=sim/sim.cpp common/id.cpp → simulator`.

---

## 5. "Backend": `backend/backend.h` — Not a Server

Despite `docs/README.md` claiming FastAPI ingestion at `POST /telemetry` @ 50Hz, reality:

* `backend/backend.h` is a **C++ header-only store**: `DroneState` (snapshot + `last_updated=now()`) + `DroneList { unordered_map<int,DroneState> }` with `addDrone O(1)`, `update O(1)`, `writeTelemetry O(N)` via manual `ofstream` string concatenation.
* No HTTP, no Python dict, no uvicorn here. Docs are aspirational/false.
* Manual JSON: no escaping of `addr`. Safe today (addresses contain no `"`), fragile tomorrow.
* `timestamp` = serialization time, identical for all 100 drones per tick — not physics time.

---

## 6. API Shim: `api/main.py` (48 lines)

```python
GET / → {"message":...}
GET /telemetry → json.load(open("../backend/telemetry.json"))
```

* CORS allows only `http://localhost:5173` (Vite default).
* `../backend/...` is CWD-relative — breaks if uvicorn launched elsewhere. No Pydantic validation, no POST endpoint, no caching. Every dashboard poll triggers full file parse `O(N)`.
* Commented test fixture (`IDLE/DELIVERING`) is stale vs real states.

True data-flow rate: `1 snapshot / 2s`, not `50 updates/s` as docs claim.

---

## 7. Dashboard: `dashboard/display/src/App.tsx` (317 lines)

Stack: React 19 + `react-leaflet@5` + `leaflet@1.9.4`, Vite.

* `fetch("http://127.0.0.1:8000/telemetry")` on mount + `setInterval(2000)` — frequency-matched to sim, so aliasing/beating possible.
* **Two-level classification** (the cleverest logic in repo):

```ts
stateColor: battery<=25 → red; CRUISE/...→blue; RETURNING→amber; else grey
normalizeState: battery<=25 → LOW_BATTERY; CRUISE/...→ACTIVE; RETURNING; else IDLE
```

This collapses 7 sim states → 4 UI buckets. `metrics = {total,active,returning,lowBattery,idle,avgBattery}` via `useMemo O(N)`.

* Map: center `[36.1699,-115.1398]` (Vegas), zoom 13, OSM tiles. `CircleMarker` per drone (100 DOM nodes — fine; 10k would need canvas/clustering). Selected drone gets `Polyline base→pos→dest` dashed + `flyTo(pos,15,0.8s)` via `FocusDrone`.
* `Tooltip` (hover) + `Popup` (click) show `id,battery,state,base,destination`.
* Styling `App.css`: dark sidebar `320px` + light topbar, critical alert banner if `lowBattery>0`.

Hardcodings: API URL, thresholds, map center — no env config.

---

## 8. Verdict: Strengths vs Load-Bearing Flaws

**Strengths:** deterministic world-gen, clean entity/store split, `O(1)` update + `O(N)` serialize is optimal for snapshot broadcast, UI bucketing is sound abstraction.

**Must-fix for correctness:**

1. Atomic write (`write tmp + rename`) to kill torn reads.
2. Unify battery threshold (20 vs 25) and speed distribution vs artifact.
3. Replace relative paths with absolute/config, add `POST /telemetry` or delete false docs.
4. Model battery-death (no flight at 0%), handle `SIGTERM`, remove dead `START/APPROACH`, `distance2D`, `chargingTicksRemaining` (written never read).
5. Validate JSON with Pydantic, escape strings, use haversine if going beyond demo.

Run order: `make && ./simulator` (root) → `uvicorn api.main:app --port 8000` (from `api/`, path fix needed) → `npm run dev` (from `dashboard/display`).
