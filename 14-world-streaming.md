# ScaleFast Engine — World System and Streaming

## Part 14: Discrete Maps, id-Style Loading, Zero Hitches

### Philosophy

Discrete maps with aggressive precomputation at build time, surgical streaming at runtime, and load times so fast they feel like transitions. Open worlds are an engineering tar pit — 80% of budget managing what the player MIGHT see instead of optimizing what they ARE seeing.

**A map is a closed, bounded, fully known dataset.** The engine knows every piece of geometry, every light, every texture at build time. This enables:

- Complete visibility precomputation
- Optimal texture packing per map
- Pre-baked SDF volumes for particle collision
- Pre-built light cluster hints
- Shadow map pre-caching for static lights
- Memory budget fully known and bounded
- Pre-compiled shader pipeline state objects

### Map Structure

```
Sectors:
  Spatial subdivisions defining streaming granularity
  Each sector: bounding box, mesh refs, light refs, entity refs

Geometry:
  Static meshes: pre-batched, sorted by material
  Collision meshes: simplified, fed to Jolt as static bodies
  SDF volumes: pre-baked per sector for particle collision
  Occlusion proxy meshes for software occlusion culling

Visibility:
  Precomputed sector-to-sector visibility sets
  Runtime occlusion culling refines conservatively

Lights:
  All defined, shadow atlas regions pre-assigned
  Static shadow maps PRE-BAKED and loaded as raw data
```

### The Loading Model

**What loads at map start (the 2-4 second load):**
- All geometry (meshes, collision, occlusion proxy)
- All shader PSOs (pre-compiled, zero runtime compilation)
- Low-res texture set (minimum quality, render immediately)
- All light data + pre-baked static shadow maps
- All audio banks
- SDF volumes, visibility data

**What streams during gameplay (invisible):**
- High-res texture mip levels only

**Geometry never streams mid-map.** It's all resident from load. Only texture quality improves. Mip-level upgrades are inherently invisible — a texture going from 512² to 2048² while you walk toward it is imperceptible.

### Load Time Breakdown

```
1. Read header, allocate pools           ~50ms
2. Parallel decompress (Zstd, NVMe)     ~500-1500ms
3. Upload geometry to GPU                ~200-400ms
4. Create Jolt static bodies             ~100-200ms
5. PSO cache validation                  ~0-100ms
6. Initialize entities, map scripts      ~50-100ms
                                         ────────
                                Total:   ~1-2.5 seconds
```

The job system parallelizes decompression — each sector is an independent job. Start rendering as soon as spawn-point sectors load; background sectors finish while the player can't see them.

### Virtual Texturing

```
Build time:
  All map textures packed into virtual texture atlas
  Full mip chains generated, BC7/BC5 compressed
  64KB pages per mip level stored in map file

Runtime:
  GPU feedback buffer: shader writes accessed page IDs + mip levels
  CPU reads feedback (1 frame latency)
  Streaming prioritizes highest-demand pages
  Pages uploaded to physical texture cache (GPU atlas)
  LRU eviction for unused pages

Worst case: slightly blurry texture for 1-2 frames
  (lower mips always resident, loaded at map start)
```

### Pre-Compiled Pipeline State Objects

Every shader permutation compiled at build time (or first-run). Stored in pipeline cache per GPU/driver combination. Zero runtime shader compilation. Zero stutter.

**UE5 comparison:** UE5's shader compilation stutter is one of its most visible failures. ScaleFast knows every material in the map at build time. All PSOs are pre-cached. This problem simply doesn't exist.

### How This Feeds Other Systems

- **Shadows:** Static shadow maps baked at build time, loaded as raw data. First frame has perfect shadows at zero runtime cost
- **Particles:** SDF volumes precomputed per sector, resident immediately
- **Lights:** Cluster hints pre-identify high-density areas for grid optimization
- **Decals:** Static decals baked into map. Only player-caused decals are runtime
