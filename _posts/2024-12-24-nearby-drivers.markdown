---
layout: post
title:  "Nearby Drivers..."
date:   2024-12-24
categories: system-design geometry uber h3
---

## Nearby Drivers Sounds Easy… Until It Isn’t

A ride-sharing app needs one core operation to be fast: **Given a rider’s location, find nearby drivers.**

The naive way is obvious:
1. Store every driver as `(lat, lon)`
2. For a rider request, compute the distance from the rider to every driver
3. Sort and pick the closest

That distance is usually computed using the [Haversine formula](https://community.esri.com/t5/coordinate-reference-systems-blog/distance-on-a-sphere-the-haversine-formula/ba-p/902128) (or something equivalent),
because Earth is curved and naive Euclidean distance gets inaccurate.

### Why the naive approach fails

If you have `N` active drivers, a single rider request does:

- `N` distance computations
- each one involves trig (`sin`, `cos`, `asin`, `sqrt`)
- then sorting or partial sorting

So the request is basically **O(N)**. With 100,000 active drivers, that’s 100,000 expensive computations *every time the app opens*. Now multiply that by thousands of concurrent riders, plus real-time driver movement.

Your database CPU will catch fire 🔥 not because SQL is bad, but because you’re forcing it to do geometry at scale.

### The Key Insight: Stop Thinking in Coordinates

Here’s the shift that makes everything click. “Nearby” is not really a geometric idea. It’s a **neighborhood**. Humans think in meters and kilometers. Databases don’t. Databases are much happier working with **sets, keys, and equality checks**.

So instead of asking: `Which drivers are within X meters of this point?`

You ask a different question: `Which drivers are in the same region as me, or in the regions right next to it?`

If you can answer that efficiently, exact distance suddenly matters a lot less. And this is the insight that unlocked Uber’s scaling problem.


### How Uber Solves Nearby Search at Scale

Uber doesn’t search using raw latitude and longitude. They first **discretize the Earth**. They divide the surface of the planet into a grid of **hexagonal cells**, then search those cells instead of doing distance math. To scale it, Uber built and open-sourced a system called **[H3](https://github.com/uber/h3)**.

At a high level, the flow looks like this:

- Split the Earth into hexagons
- Map every `(lat, lon)` to one hexagon
- Represent that hexagon with a 64-bit integer
- Search nearby hexagons instead of computing distances

So, no trigonometry; no per-row geometry; just fast, indexed lookups.

### From Coordinates to Cell IDs

This is where things really change. With H3, every location on Earth maps to a **single integer** at a chosen resolution. Normally, you’d store a driver’s location as two floating-point numbers: latitude and longitude. Something like `37.7749 and −122.4194`. Precise, but expensive to work with at scale. With H3, that same location becomes one value: `8928308280fffff`

That number is the **cell ID**. It represents a hexagon on the Earth’s surface that contains the point.

Resolution controls how big those hexagons are:

- lower resolution means larger hexagons (city-scale)
- higher resolution means smaller hexagons (street-level)

You choose the resolution based on what “nearby” should mean for your app. Once you do this, location stops being geometry and starts being **indexable data**. Instead of comparing distances between points, you’re comparing integers. Also, a driver  And databases are extremely good at that. Eventually, when a driver moves, nothing changes as long as they’re still inside the same hexagon. Only when they cross into a new hexagon do they update a single value: their `cell_id` (a simple key-value update). That design keeps database writes minimal and simple, avoiding constant location updates and letting the system scale under heavy movement.

I am gonna show a code example: a minimal case showing how a latitude/longitude pair is converted into an H3 cell ID.

```python
import h3
lat, lon = 37.7749, -122.5154
res = 9 # H3 resolution: controls hexagon size (higher = smaller, more precise)

cell = h3.latlng_to_cell(lat, lon, res)
print("cell:", cell) #cell: 89283095967ffff
```

This `cell` value is the H3 index. It uniquely identifies the hexagon that contains the given latitude and longitude at resolution 9.

### The Search: Nearby Without Distance

This is where the real win shows up. When a rider opens the app, Uber does **NOT** calculate distances to drivers.

Instead, the process looks like this:

1. Convert the rider’s location to an H3 cell ID  
2. Find the neighboring cells around it  
3. Fetch drivers whose cell IDs fall in that set  

Under the hood, the question becomes: `Give me all drivers in my cell, plus the drivers in the 6 neighboring cells.`

In H3 terms, this is called a **k-ring** query.

For `k = 1`, you get:
- the rider’s cell  
- its 6 immediate neighbors  

That’s a fixed, tiny set.

Conceptually, the query becomes:

```sql
SELECT *
FROM drivers
WHERE cell_id IN (
  user_cell, neighbor_1, neighbor_2, neighbor_3, neighbor_4, neighbor_5, neighbor_6);
```

This is no longer geometry. It’s a **discrete set lookup**. Databases are exceptionally good at this.

### Why Hexagons, Not Squares?

At first glance, squares feel natural. Grids are easy. But squares have an annoying property: **neighbors aren’t equal**.  
A square has 4 side-neighbors and 4 diagonal-neighbors, and the diagonal ones are farther away. So the moment you say “search one step out,” you’ve already created weird distance inconsistencies.

Hexagons fix that. Each cell has **exactly 6 neighbors**, and they’re all the same “step” away. That makes ring-expansion clean and symmetric: `k = 1` really means “one hop in any direction,” with no special cases.

There’s another practical win: hexagons approximate circles better than squares at the same scale, so a k-ring looks more like a radius search. That means fewer edge-case gaps and fewer “why didn’t it find drivers right there?” moments.

### How to expand the Search Radius

Need a wider search area? You don’t change any math. You just increase `k`.
- `k = 1` → immediate neighbors  
- `k = 2` → neighbors of neighbors  
- `k = 3` → a larger ring  

Each step adds more cells, but the growth is controlled and predictable. You’re still querying a small, bounded set of integers, not scanning the entire driver table. The naive approach scales with the **number of drivers**. This approach scales with the **number of cells**, a constant.

Instead of: `O(N) distance computations per request`

You get: `O(1) set lookups with indexes` 

That’s the difference between a system that works in a demo and one that survives real traffic.

### What I'am About to Illustrate

I’ll walk through a small, concrete simulation to show why Uber’s H3-based approach scales and the naive distance-based approach doesn’t.

I’ll start with two hexagonal regions (`R1` and `R2`) and two drivers, with a rider located in `R1`. First, I’ll compute distances the naive way by checking every driver, regardless of region. Then I’ll switch perspectives. I’ll organize drivers by H3 cell using a simple hash map (`cell_id → drivers`) and show how “nearby search” becomes a lookup over a small, fixed set of cells.

Finally, I’ll simulate driver movement across a hexagon boundary and show how the update is just a lightweight key-value change, while the nearby-search logic stays exactly the same. The goal is to make the abstraction shift obvious: from continuous geometry and full scans to discrete regions and constant-time lookups.


#### Step 1 — Set up two regions (R1, R2) and initial positions
- We’ll create **two H3 cells**: `R1` and `R2`.
- Place a **rider** in `R1`.
- Place **two drivers**: `D1` starts in `R1`, `D2` starts in `R2`.

```python
import h3
import math

# --- Helper: Haversine distance in km (naive geometry approach) ---
def haversine_km(lat1, lon1, lat2, lon2):
    # Earth radius in km
    R = 6371.0
    # Convert degrees to radians
    p1, p2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlmb = math.radians(lon2 - lon1)
    a = math.sin(dphi/2)**2 + math.cos(p1) * math.cos(p2) * math.sin(dlmb/2)**2
    return 2 * R * math.asin(math.sqrt(a))

# --- Choose a resolution (higher = smaller hexagons) ---
res = 9  # controls hex size; higher = smaller, more precise cells
# Rider is in/near San Francisco (we’ll treat this as "Region R1")
rider = {"id": "RIDER", "lat": 37.7749, "lon": -122.4194}
R1 = h3.latlng_to_cell(rider["lat"], rider["lon"], res)  # rider's cell

# Pick a second point likely to land in a different nearby cell ("Region R2")
p2_lat, p2_lon = 37.7849, -122.4094
R2 = h3.latlng_to_cell(p2_lat, p2_lon, res)

# Two drivers: D1 starts in R1, D2 starts in R2
D1 = {"id": "D1", "lat": 37.7752, "lon": -122.4188, "cell": R1}
D2 = {"id": "D2", "lat": p2_lat,   "lon": p2_lon,   "cell": R2}

print("R1 cell:", R1)               #R1 cell: 89283082803ffff
print("R2 cell:", R2)               #R2 cell: 89283082aa7ffff
print("D1 cell:", D1["cell"])       #D1 cell: 89283082803ffff
print("D2 cell:", D2["cell"])       #D2 cell: 89283082aa7ffff
```

#### Step 2 — Naive distance-based search (the slow way)

- Compute the distance from the rider (in `R1`) to **each driver**, regardless of region.
- This mimics the naive approach: check *every driver* using geometry.
- Even with just two drivers, the pattern is clear: distance math runs per driver.

In a real system, this scales linearly with the number of active drivers.

```python
# --- Naive approach: compute distance from rider to every driver ---
drivers = [D1, D2]
for d in drivers:
    dist = haversine_km(
        rider["lat"], rider["lon"],
        d["lat"], d["lon"]
    )
    print(f"Distance from rider to {d['id']}: {dist:.3f} km")
```

```text
Distance from rider to D1: 0.062 km
Distance from rider to D2: 1.417 km
```
**What this shows:**
- We do a distance calculation for **each driver**.
- Each calculation involves trigonometry (`sin`, `cos`, `asin`, `sqrt`).
- With N drivers, this is O(N) work *per rider request*.

This is fine for two drivers. It breaks down completely when N is large.


#### Step 3 — Store drivers by cell using a hash map (dict)

- Replace “scan all drivers” with an index: `cell_id -> list_of_drivers`.
- This is the core trick: once locations are discretized into cells, “nearby search” becomes a dictionary lookup.

Think of it like a database index, but in Python.

```python
from collections import defaultdict
# --- Build a hash map: cell_id -> [drivers in that cell] ---
drivers_by_cell = defaultdict(list)

for d in [D1, D2]:
    drivers_by_cell[d["cell"]].append(d)

# Quick sanity check: show how many drivers are in each region
print("Drivers in R1:", [d["id"] for d in drivers_by_cell[R1]])
print("Drivers in R2:", [d["id"] for d in drivers_by_cell[R2]])
```

```text
Drivers in R1: ['D1']
Drivers in R2: ['D2']
```

**What this shows:**
- We can now answer: “who’s in cell X?” in O(1) time (amortized).
- No trig, no geometry, no scanning.


#### Step 4 — “Nearby” search via k-ring: only check drivers in nearby cells

- Compute the rider’s cell (`R1`) and the neighbor cells around it using H3.
- Fetch candidate drivers using the hash map:
  - `drivers_by_cell[user_cell]` plus `drivers_by_cell[neighbor_cells]`.
- Only after we have a small candidate set do we compute distances (optional).

This is the big win: the database work becomes a tiny set lookup.

```python
# --- Get the rider's cell (already computed as R1) ---
user_cell = R1
# --- k-ring: user cell + its neighbors (k=1 => 7 cells total) ---
nearby_cells = h3.grid_disk(user_cell, 1)
# --- Candidate drivers = union of drivers in these cells ---
candidates = []
for c in nearby_cells:
    candidates.extend(drivers_by_cell.get(c, []))

print("Nearby cells count:", len(nearby_cells))
print("Candidate drivers:", [d["id"] for d in candidates])

# Optionally compute distances only for CANDIDATES (small set)
for d in candidates:
    dist = haversine_km(rider["lat"], rider["lon"], d["lat"], d["lon"])
    print(f"Candidate distance to {d['id']}: {dist:.3f} km")
```

```text
Nearby cells count: 7
Candidate drivers: ['D1']
Candidate distance to D1: 0.062 km
```

**What this shows:**
- We’re not scanning all drivers anymore.
- We’re scanning only drivers in a small, local neighborhood of cells.
- Distance math becomes optional and cheap because the candidate list is small.


#### Step 5 — Simulate movement: D1 crosses from R1 to R2 (cheap update)

- If a driver moves *within the same cell*, nothing changes (no update needed).
- If a driver crosses into a new cell, we do a simple “remove from old list, add to new list”.
- Then run the same k-ring search again and see how results change.

This is the “key-value update” idea in action.

```python
# --- Simulate D1 moving to the coordinates of R2 (crossing a cell boundary) ---
old_cell = D1["cell"]
# New GPS position for D1 (we'll reuse p2_lat/p2_lon so it lands in R2)
D1["lat"], D1["lon"] = p2_lat, p2_lon
new_cell = h3.latlng_to_cell(D1["lat"], D1["lon"], res)
print("D1 old cell:", old_cell)
print("D1 new cell:", new_cell)

# Only update the index if the cell actually changed
if new_cell != old_cell:
    # Remove D1 from old cell bucket
    drivers_by_cell[old_cell] = [d for d in drivers_by_cell[old_cell] if d["id"] != "D1"]
    # Add D1 to new cell bucket
    D1["cell"] = new_cell
    drivers_by_cell[new_cell].append(D1)

print("Drivers in R1 after move:", [d["id"] for d in drivers_by_cell[R1]])
print("Drivers in R2 after move:", [d["id"] for d in drivers_by_cell[R2]])

# --- Run the same nearby search again (rider still in R1) ---
nearby_cells = h3.grid_disk(R1, 1)

candidates_after = []
for c in nearby_cells:
    candidates_after.extend(drivers_by_cell.get(c, []))
print("Candidate drivers after D1 move:", [d["id"] for d in candidates_after])

# Optional: distances after movement (still only for candidates)
for d in candidates_after:
    dist = haversine_km(rider["lat"], rider["lon"], d["lat"], d["lon"])
    print(f"After move distance to {d['id']}: {dist:.3f} km")
```

```text
D1 old cell: 89283082803ffff
D1 new cell: 89283082aa7ffff
Drivers in R1 after move: []
Drivers in R2 after move: ['D2', 'D1']
Candidate drivers after D1 move: []
```

**What this shows:**
- Movement updates are **sparse**: most GPS pings don’t change the cell, so no write.
- When the cell changes, the update is just moving one driver between two buckets.
- The “nearby” query stays the same — it always queries `R1 + neighbors`.


#### Step 6 — Expanding the search radius (correct behavior)

- Start searching from the rider’s own cell (`k = 0`)
- Gradually expand outward until drivers are found
- Stop once enough candidates are discovered
- Never “lose” candidates found at smaller k

This fixes a subtle but important logic issue in naive implementations.

```python
# --- Expand search radius until we find enough drivers ---
min_candidates = 2   # minimum drivers we want
k_max = 8            # cap to keep latency bounded
final_candidates = []
seen = set()
for k in range(0, k_max + 1):
    cells = h3.grid_disk(R1, k)
    candidates = []
    for c in cells:
        for d in drivers_by_cell.get(c, []):
            if d["id"] not in seen:
                seen.add(d["id"])
                candidates.append(d)
    print(f"k={k}: found {len(candidates)} candidate drivers")
    # Keep what we found so far
    if candidates:
        final_candidates = candidates
    # Stop early once we have enough total drivers
    if len(seen) >= min_candidates:
        break

if not final_candidates:
    print("No drivers available nearby.")
else:
    # Exact distance math only on the final small set
    final_candidates.sort(
        key=lambda d: haversine_km(
            rider["lat"], rider["lon"],
            d["lat"], d["lon"]))
    print("Top candidates:", [d["id"] for d in final_candidates[:min_candidates]])
```

```text
k=0: found 0 candidate drivers
k=1: found 0 candidate drivers
k=2: found 0 candidate drivers
k=3: found 0 candidate drivers
k=4: found 0 candidate drivers
k=5: found 2 candidate drivers
Top candidates: ['D2', 'D1']
```

**Why this matters:**
- Search radius only grows when needed
- Candidates found at smaller k are preserved
- Work stays proportional to area, not driver count
- Behavior matches real-world ride-dispatch systems

At scale, the trick is rarely faster math. It’s choosing the right abstraction. Uber turns continuous geometry into discrete indexing, so “nearby” becomes a small set lookup instead of a city-wide scan. That’s what keeps latency predictable and databases alive under real traffic.

### References

1. [Uber Engineering — H3: A Hexagonal Hierarchical Spatial Index](https://eng.uber.com/h3/)
2. [H3 source code](https://github.com/uber/h3)

---