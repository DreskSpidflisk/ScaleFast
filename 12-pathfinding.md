# ScaleFast Engine — Pathfinding and AI Navigation

## Part 12: NavMesh, Dynamic Obstacles, and AI at Scale

### Navigation Meshes

NavMeshes are built at map build time from static collision geometry. Multiple navmesh layers per map for different agent types:

```
Humanoid:      1.8m height, 0.3m radius, 45° max slope
Large creature: 3.0m height, 1.0m radius, 35° max slope
Small creature: 0.5m height, 0.15m radius, 60° max slope
Vehicle:       1.5m height, 2.0m radius, 20° max slope, min turn radius
```

Pathfinding: A* over navmesh adjacency graph, funnel algorithm for shortest path within triangle corridor. Typically < 0.1ms per path.

### Dynamic Obstacle Handling

**Layer 1: Local Avoidance (per-frame, handles most obstacles)**

Each AI casts 3-5 short detection rays in its movement direction via Jolt raycasts. These see ALL geometry including dynamic rigid bodies. AI steers around single crates, NPCs, closing doors without involving the pathfinding system.

50 AIs × 5 raycasts = 250 raycasts/frame. Jolt handles thousands per ms.

**Layer 2: NavMesh Obstacle Carving (periodic, handles blockages)**

Dynamic rigid bodies above a size threshold register as navmesh obstacles. Their footprint is projected onto the navmesh, marking triangles as temporarily non-traversable. A* skips blocked triangles.

When a path is blocked: AI waits 0.5-1.0s (obstacle might move), then repaths. If no path exists, AI enters designer-configured "blocked" behavior state.

### Path Cache (Horde AI Optimization)

Multiple AIs targeting the same area compute nearly identical paths. Cache recent A* results:

```
Key:   (start_region, goal_region, agent_type)
Value: triangle corridor (A* result before string-pulling)

Cache hit: skip A*, do string-pulling only (unique per AI, cheap)
50 AIs all pursuing the player: 1 A* search + 49 cache hits.
Cost reduction: ~98%.
```

### Shared Sensing

**Visibility cache:** Spatial hash of recent "can X see Y?" queries. Nearby AIs reuse results. Invalidated every 0.25-0.5s.

**Raycast batching:** All AIs submit raycast requests to a central queue. Deduplicate similar rays. Single Jolt batch query. 250 rays → ~80-100 unique rays after dedup.

### Flying Enemies

Hybrid approach: sparse 3D waypoint graph for global navigation (designer-placed at transition points — doorways, vertical shafts) + steering behaviors for local movement (maneuver within rooms, dodge, circle).

### Vehicle Pathfinding

Modified A* on vehicle navmesh layer, post-processed with Dubins curves (forward-only, minimum turn radius) or Reeds-Shepp curves (allows reversing). PID controller follows computed curve. Jolt's vehicle constraint handles physics.

### AI Job System Integration

```
Job: AIPerception    — batched raycasts, visibility cache    ~0.3ms
Job: AIDecision      — behavior trees, state machines         ~0.1ms
Job: AIPathfinding   — A* for repath requests, cache lookup   ~0.2ms
Job: AISteering      — desired velocity, avoidance blend      ~0.1ms
Job: NavMeshCarving  — update blocked triangles               ~0.1ms
                                                              ─────
Total for 50 AIs:                                             ~0.8ms
```

### AI LOD (Distance-Based Update Rate)

```
Close (< 20m):   full update every frame
Medium (20-50m): update every 2nd frame, reduced raycasts
Far (50-100m):   update every 4th frame, simplified steering
Very far (100m+): update every 8th frame, interpolated movement
```

500 AIs with LOD: effective per-frame cost of ~120 full AIs. ~1.5ms.
