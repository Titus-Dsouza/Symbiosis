# Intelligent Route & Load Optimization for Logistics Fleets — Roadmap

## Architecture

```
Frontend (React + Leaflet map)
        │  REST/JSON
Backend (FastAPI)
        │
   ┌────┴────┬──────────────┬─────────────┐
Optimizer   Distance      Data/Store    Disruption/
(OR-Tools)  (OSRM/haversine) (in-memory) Tracking/Reroute
```

## Python libraries (backend)

| Purpose | Library | Why |
|---|---|---|
| API server | **FastAPI** + **uvicorn** | fast to stand up, auto docs at `/docs`, easy CORS for a separate frontend |
| Route/load optimizer (the AI core) | **OR-Tools** (`ortools.constraint_solver`) | Google's free constraint solver, built-in support for Capacitated VRP + Time Windows + per-vehicle cost/capacity — this *is* the "AI system" judges will care about |
| Data validation/models | **Pydantic** | request/response schemas for vehicles, orders, routes |
| Road distances | **requests** → free public **OSRM** demo API (no key) | real road-network km/min instead of straight-line guesses; haversine formula (pure math, no lib) as offline fallback if OSRM is rate-limited during the demo |
| Mock data | stdlib `random`/`math` | seed vehicles/owners/orders around one city so the map looks realistic |

No paid API keys anywhere — OSRM's public server and OpenStreetMap tiles are free.

## Frontend

- **React + Vite** — fast setup, no build config pain
- **Leaflet.js + OpenStreetMap tiles** — free map rendering, no API key (unlike Google Maps)
- Plain fetch calls to the FastAPI backend, no heavy state library needed for a hackathon scope

## Problem framing

This is a **Heterogeneous Fleet Capacitated Vehicle Routing Problem with Time Windows (HFVRPTW)**, plus a live fleet-management/disruption layer on top:
- Heterogeneous fleet: 3-wheeler / mini-truck / truck, each with its own capacity, fuel cost/km, emissions/km, driver cost/hr, and average speed.
- Capacitated: each vehicle can't carry more than its capacity in kg.
- Time windows: each delivery has an allowed delivery window.
- Multi-stop: a single vehicle serves several orders in one route.
- Objective: minimize total cost (fuel + toll + driver time) and emissions per trip, versus a naive "no optimization" baseline, to produce a headline cost/emissions savings %.

## Roles & interfaces

Four separate views on top of the same backend, gated by a lightweight login (username + role picker, **no password security** — this is a hackathon demo, not a production auth system).

| Role | Sees | Can do |
|---|---|---|
| **Fleet Manager / Dispatcher** | Full fleet map, all routes, all orders, cost/emissions dashboard | Run optimizer, full CRUD on vehicles/owners, trigger/resolve breakdowns and reroutes |
| **Vehicle Owner** | Only their own vehicles, current status, assigned route, revenue earned | Nothing fleet-wide — read-only on their own vehicles |
| **Driver** | Their one assigned vehicle's route for the day (ordered stop list + map) | Mark a stop delivered, report a breakdown (calls the same `/disrupt` the dispatcher uses) |
| **Customer** | Their own order's status + live (simulated) position of the assigned vehicle | Look up by order id + phone, read-only |

Login flow: `POST /login {username, role}` → returns a session token plus, for Owner/Driver, the `owner_id`/`driver_id` it's scoped to. All role-scoped endpoints (`/my-vehicles`, `/my-route`, `/track-order`) filter server-side by that id — the frontend never just hides data client-side.

New data needed: a `Driver` entity (id, name, phone, assigned `vehicle_id`), and `customer_name`/`customer_phone` fields added to `Order` so customers have something to look up by.

Frontend stays a single React app (shared map/API code) with React Router sending each role to its own dashboard after login, rather than four separate apps.

## Feature coverage map

Every bullet from the problem statement, mapped to concrete tasks — nothing gets silently dropped. Anything not fully solved (given zero budget / no live telemetry) is explicitly marked **[simulated]** rather than omitted.

| # | Problem statement feature | How it's implemented | Status |
|---|---|---|---|
| 1 | **Vehicle assignment** — type of vehicle, capacity | `VehicleType` model (3-wheeler/mini-truck/truck) with capacity_kg; OR-Tools assigns orders to the right vehicle type via per-vehicle capacity dimension | Core |
| 2 | **Load optimization** — remaining weight | `load_used_kg` vs `load_capacity_kg` returned per route; UI shows a fill-% bar per vehicle so under-utilized capacity is visible | Core |
| 3a | Fuel cost | `fuel_cost_per_km` per vehicle type, summed per route | Core |
| 3b | Toll tax | Flat toll-per-km applied to toll-eligible vehicles on legs over a distance threshold | Core |
| 3c | Driver cost | `driver_cost_per_hour` × route travel time | Core |
| 3d | Price per kg (different types) | `price_per_kg` per order → revenue per route, shown alongside cost so margin is visible | Core |
| 3e | **Tracking** | Vehicle position simulated by interpolating along its assigned route based on elapsed time since dispatch; polled by frontend and animated on the map | **[simulated]** — no real GPS hardware, zero budget |
| 3f | **Route redirectioning** | `/reroute/{vehicle_id}` endpoint: simulate a blocked/slow road segment, mark that edge as high-cost in the distance matrix, re-solve just that vehicle's remaining stops | New — added in Day 3 |
| 4 | **Vehicle, owner, detail** | `Vehicle`/`Owner` models with full CRUD (create/read/update/delete, not just create+list); a detail view per vehicle showing owner, plate, type, current status, current route; Owner gets their own scoped dashboard (see Roles & interfaces) | Upgraded from "create+list" to full CRUD + own dashboard |
| 5 | **Multistop** [Difficult] | Native output of the VRP solver — each vehicle route already visits multiple orders in solver-chosen order | Core |
| 6a | **Breakdown inform** | `/disrupt/{vehicle_id}`: marks vehicle unavailable, re-optimizes remaining orders across rest of fleet, returns a diff (which orders moved to which vehicle); Driver role can trigger this themselves from their own dashboard | Core |
| 6b | **Empty returns** | Each route reports `return_leg_km` (last stop → depot) and its cost/emissions; a `/backhaul-check` step looks for any unassigned order within a small detour of that return leg and appends it if it fits capacity/time window | Upgraded from "flag only" to an actual backhaul attempt |

## Roadmap (multi-day budget)

### Day 1 — Core optimizer (backend only, test via `/docs` or curl)
1. Models: Vehicle, VehicleType, Order, Owner (feature 1, 2, 4 data layer).
2. Mock data generator (depot + fleet + orders around one city).
3. Distance module: OSRM table API with haversine fallback.
4. Optimizer: OR-Tools HFVRPTW — per-vehicle cost callback (fuel+toll), capacity dimension, time-window dimension, disjunctions to allow dropping infeasible orders (features 1, 2, 3a-3d, 5).
5. Naive baseline calculator (bin-pack + visit-in-input-order) → "% cost/emissions saved by AI" headline number.
6. FastAPI endpoints: `/vehicles`, `/owners`, `/orders`, `/optimize`.

### Day 2 — Full CRUD + auth + Fleet Manager frontend
7. Full CRUD on `/vehicles` and `/owners` (create/update/delete, not just create+list) + vehicle detail endpoint (feature 4).
8. `Driver` model, `customer_name`/`customer_phone` on `Order`; `POST /login {username, role}` returning a scoped session.
9. Leaflet map: depot pin, vehicle markers, optimized routes as colored polylines per vehicle.
10. Route/cost summary panel: per-vehicle load %, cost breakdown, revenue, total emissions, optimized-vs-baseline savings %.
11. Fleet Manager dashboard (the full/admin view): fleet map, optimizer trigger, vehicle/owner tables + detail drawer.

### Day 3 — Disruption, reroute, tracking, empty returns
12. `/disrupt/{vehicle_id}` — breakdown → re-optimize remaining fleet live (feature 6a); map animates the change.
13. `/reroute/{vehicle_id}` — simulate a blocked road segment, re-solve that vehicle's remaining stops around it (feature 3f).
14. Tracking simulation — backend computes each vehicle's current interpolated position along its route by elapsed time; frontend polls and animates marker movement (feature 3e).
15. Backhaul/empty-return check on each route's return leg (feature 6b).

### Day 4 — Owner, Driver, Customer dashboards
16. Role-scoped endpoints: `/my-vehicles` (Owner), `/my-route` (Driver), `/track-order` (Customer, looked up by order id + phone).
17. Vehicle Owner dashboard: own vehicles, status, revenue per trip — reuses map/summary components from Day 2, filtered.
18. Driver dashboard: today's assigned stop list + mini-map, "mark delivered" and "report breakdown" buttons (breakdown button calls `/disrupt` for their own vehicle).
19. Customer tracking page: order status + simulated live position of their vehicle, no login required beyond order id + phone.
20. Login screen + React Router wiring to send each role to its own dashboard.

### Day 5 — Demo polish
21. Seeded scenario tuned to reliably show a strong before/after savings number.
22. Demo script walking through all 4 roles and all 6 features in order, including a driver-reported breakdown and a live reroute.

## Cut list if time runs short (in priority order to cut)

1. Customer tracking dashboard — fall back to a `/track-order` API only, no dedicated UI.
2. Vehicle Owner dashboard — fall back to the Fleet Manager view filtered by owner as a query param, no separate role.
3. Backhaul optimization on empty returns — fall back to reporting empty-return cost/emissions without trying to fill it.
4. Reroute-around-blocked-segment — fall back to breakdown-style re-optimization only.
5. Tracking animation — fall back to static "last known position" markers, refreshed on each optimize/disrupt call.
6. Full CRUD — fall back to create + list only.

The Fleet Manager dashboard and Driver breakdown-report flow are never cut — between them they demonstrate every core feature plus the multi-role angle.

Multistop, vehicle assignment, load optimization, and cost/emissions optimization (features 1, 2, 3a-3d, 5) are never cut — they're the solver core the whole pitch depends on.

## Open decisions to revisit

- Exact vehicle type roster and cost/emission numbers (currently placeholders: 3-wheeler, mini-truck, truck)
- Demo city/coordinates for seeded data
- How aggressively to attempt real backhaul optimization for feature 6b given it's the hardest remaining piece
