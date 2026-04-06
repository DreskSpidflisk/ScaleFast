# ScaleFast Engine — Networking

## Part 13: Server-Authoritative Multiplayer with Synchronized Physics

### Architecture: Dedicated Server, Client-Predicted

```
Server: authoritative simulation (physics, AI, IK, game logic)
Client: predicted local movement, interpolated remote entities
Trust: server validates everything, client trusted for nothing
```

### Transport

UDP with custom reliability layer:
- **Unreliable channel:** position snapshots, physics state (lost packet superseded by next)
- **Reliable-ordered channel:** game events (limb severed, door opened, objective captured)
- **Reliable-unordered channel:** asset streaming, chat

### Server Tick (60 Hz)

Each tick:
1. Receive client inputs (timestamped)
2. Process inputs in tick order
3. Run simulation (physics, AI, physics-coupled IK, game logic)
4. Build world snapshot
5. Delta-compress against each client's last ACK'd snapshot
6. Send per-client deltas

### Client-Side Prediction

Local player input is applied immediately — zero perceived latency. The client runs the exact same character controller code as the server (same Jolt physics, same character capsule). Given identical input and state, the result is identical (Jolt is deterministic).

**Why corrections are rare:** Movement is deterministic given input. The client has correct world geometry. The only misprediction source: other entities affecting the local player (explosions, being shot, collision). These are rare compared to frames of normal movement. 95%+ of frames: prediction is perfect.

**When correction is needed:**
1. Snap internal state to server's authoritative position at tick T
2. Replay all inputs from tick T to current tick (from stored history)
3. Visual difference blended over 2-8 frames in presentation layer
4. Simulation snaps immediately; rendering interpolates smoothly

This is why Overwatch feels perfect — same architecture, same principle. The client never perceives hard teleports even during large corrections.

### Remote Entity Interpolation

Other players are NOT predicted. They are interpolated between server snapshots:

```
render_time = current_time - interpolation_delay (2 × tick interval ≈ 33ms)
position = lerp(snapshot_T, snapshot_T+1, alpha)
```

Perfectly smooth, no jitter, no skipping. Cost: you see them 33ms in the past (invisible in gameplay at 60 tick/s). Dropped packet: extrapolate from last known velocity until next packet. One drop = ~16ms extrapolation. Imperceptible.

### Synchronized Rigid Bodies (Breaking the Industry Stigma)

**The industry says:** ragdolls must be client-side. **ScaleFast says:** the designer decides.

**Three sync modes:**

| Mode | Use Case | Bandwidth |
|------|----------|-----------|
| **ServerAuthoritative** | Doors, elevators, vehicles, gameplay-critical props | Full state per tick when active |
| **ImpulseSync** | Ragdolls, physics props, barrels, crates | Impulse events + periodic corrections |
| **ClientSide** | Debris, shell casings, cosmetic physics | Zero networking |

**Engine-enforced rule:** If `blocks_projectiles` or `affects_players` is true, sync mode MUST be ServerAuthoritative or ImpulseSync. The engine rejects ClientSide for gameplay-affecting bodies. A crate that blocks bullets must be synchronized.

### Ragdoll Synchronization (ImpulseSync)

Jolt's cross-platform determinism enables impulse-based sync:

**Death event (reliable, one-time):**
- Character ID, death impulse (direction + force), timestamp
- Client activates ragdoll at server tick, applies impulse locally
- Deterministic physics → nearly identical ragdoll trajectory

**Periodic correction (every 0.5s):**
- Server sends root bone position + velocity
- Client compares, smoothly blends if diverged (over 0.2s)

**Additional impulses (event-based):**
- Ragdoll kicked/shot/exploded → server sends impulse event
- Client applies, both simulate deterministically from there

**At rest:**
- Server sends final pose (all bones, ~400 bytes one-time)
- Client snaps, puts ragdoll to sleep

**Total bandwidth per ragdoll lifetime:** ~640 bytes.
Compare to full state sync: 25.2 KB/s continuous.

### Hit Detection (Valve-Style Lag Compensation)

```
Client: fires at what they SEE (sends tick + aim direction)
        Does NOT apply damage locally
        Plays muzzle flash, sound, tracer immediately (cosmetic)

Server: receives fire command at tick T
        Rewinds all hitboxes to tick T positions (from history buffer)
        Casts ray against rewound hitboxes
        Hit confirmed → apply damage at current tick
        Hit rejected → client was desynced or cheating
```

The server never trusts the client's claim of "I hit X." It verifies independently. Cheat sending "I headshot everyone" → server checks → rejects all impossible hits.

**Server history buffer:** 60 ticks × 34 entities × 30 bytes = ~61 KB. Trivial.
**Maximum rewind:** capped at 200-250ms (configurable). Beyond that, player latency is too high for fair compensation.

**Server-side fire culling:** Server does quick broad-phase check before full rewind. No entities near the ray? Skip rewind. 90% of shots miss — most skip the expensive rewind.

### Bandwidth Budget

```
8-player match, 30 AI:

Per client downstream:
  Player states:     ~8 KB/s  (7 players, delta compressed)
  AI states:         ~20 KB/s (30 enemies, sleep + delta)
  Synced rigid bodies: ~2-5 KB/s (impulse-based)
  Game events:       ~1-3 KB/s
  Total:             ~31-36 KB/s per client (~280 Kbps)

Per client upstream:
  Input packets:     ~6 KB/s (redundant last 3 inputs per packet)
  Fire events:       ~1-2 KB/s
  Total:             ~7-8 KB/s (~64 Kbps)

Server outbound (8 clients):
  ~288 KB/s (~2.3 Mbps)
```

### Server CPU Budget

```
Physics (Jolt):           ~2.0ms
AI (30 enemies):          ~0.8ms
Physics-coupled IK:       ~0.3ms
Character controllers:    ~0.2ms
Game logic:               ~0.5ms
Networking (serialize):   ~0.3ms
                          ──────
Total:                    ~4.1ms per tick (60 Hz = 16.67ms available)
Headroom:                 ~12.5ms
```

No GPU needed on server. Multiple match instances per server machine viable.

### Scaling

- **2-8 players:** trivial, everything works as described
- **8-16 players:** dedicated server preferred, AI may reduce
- **32+ players:** area-of-interest filtering, variable tick rates per distance, dedicated networking optimization work
- **64+ players:** fundamentally different architecture needed (not initial target)

**UE5 comparison:** UE5's networking is built for convenience with replication macros and automatic property sync. ScaleFast is explicit — the designer marks what gets networked, chooses the sync mode, and sees the bandwidth cost. UE5's approach is easier to use but harder to optimize. ScaleFast's approach requires more designer awareness but produces tighter, more predictable networking.
