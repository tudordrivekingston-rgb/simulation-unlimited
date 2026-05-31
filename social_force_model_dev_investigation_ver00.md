# Social Force Model (Helbing) — Development Investigation v0.0

**Date:** 2026-05-31
**Author:** LittleGreenie 🐾
**Base Repo:** [simulation-unlimited](https://github.com/robertwaltham/simulation-unlimited)
**Target:** Implement a real-time Social Force Model (Helbing & Molnar, 1995) pedestrian simulation

---

## 1. Goal

Build an interactive, real-time crowd simulation using the Social Force Model on Apple GPU via Metal compute shaders. The base repo provides a fully functional Metal compute pipeline; this investigation assesses how much of the existing code can be reused and what needs to be written from scratch.

---

## 2. Codebase Architecture

All simulations (Slime, Boids, Particle Life, Sand) follow the same pattern:

### Pattern
```
SwiftUI View → MTKView (UIViewRepresentable)
                → ViewModel (config structs, touch handling)
                → Coordinator (MTKViewDelegate)
                    → Metal pipeline setup
                    → Per-frame: compute → blur → render
                → .metal shaders (simulation logic on GPU)
```

### Key Components (Boids — closest to SFM needs)

| File | Purpose |
|---|---|
| `BoidsView.swift` | SwiftUI → MTKView bridge, Coordinator class with full Metal pipeline |
| `BoidsViewModel.swift` | Config structs (`BoidsConfig`, `BlurConfig`), touch management |
| `BoidShaders.metal` | GPU kernels: `simulateBoids`, `drawBoids`, `boxBlurBoids`, `clearTexture`, `copyToOutput` |
| `ShaderTypes.h` | Shared C/Metal types: `Particle`, `Obstacle`, buffer indices, config structs |

### Render Pipeline (per-frame)
1. `clearTexture` — blank the output texture
2. `simulateBoids` — compute new positions (GPU particle buffer update)
3. `drawBoids` — write particle positions to texture
4. `boxBlurBoids` — soften/smear trails
5. `copyToOutput` — final blit to drawable
6. *(Optional)* Render pipeline for 3D triangle rendering

Particles are stored in an `MTLBuffer` on GPU memory. The CPU reads back positions via `extractParticles()` for debug purposes.

---

## 3. Social Force Model Requirements

### Helbing Model Equations

**Main equation:**
```
m_i * dv_i/dt = F_goal_i + ΣF_repel_ij + ΣF_wall_iw + ΣF_attract_ik + ξ_i
```

**1. Goal force:**
```
F_goal_i = m_i * (v0_i * e_i - v_i) / τ
```
where `v0_i` = desired speed, `e_i` = unit vector toward goal, `v_i` = current velocity, `τ` = relaxation time

**2. Pedestrian repulsion:**
```
F_repel_ij = A_i * exp((r_ij - d_ij) / B_i) * n_ij
```
where `A_i` = interaction strength, `B_i` = falloff range, `r_ij` = sum of radii, `d_ij` = distance, `n_ij` = normalized direction

**3. Wall repulsion:**
```
F_wall_iw = A_w * exp((r_i - d_iw) / B_w) * n_iw
```
where `d_iw` = distance to wall, `n_iw` = wall normal

**4. Attraction forces:** *(optional for v1)*
```
F_attract_ik = C_ik * attraction strength
```

**5. Fluctuation:** `ξ_i` = random perturbation

### Key Properties
- **O(n²)** complexity — each pedestrian interacts with all others
- **Parallelizable** — per-particle computations are independent (read all, write one)
- **Embarrassingly GPU-friendly** — each thread handles one pedestrian

---

## 4. Mapping: Existing → SFM

### Directly Reusable ✅

| Existing Component | How It Maps to SFM |
|---|---|
| `Particle` struct (Boids) | Extend with `goal_pos`, `desired_speed`, `radius` |
| `Obstacle` struct (point repulsion) | Base for wall segment repulsion |
| GPU `device` particle buffer | Same buffer pattern — position, velocity, acceleration |
| Config buffer system (`BoidsConfig`) | Replace with `SFMConfig` struct |
| Thread dispatch (`threadCount(arrayCount:)`) | Reuse — one thread per pedestrian |
| Compute pipeline setup | Reuse — `makeComputePipelineState` |
| Texture management (path textures, blur) | Reuse — trail/blur is useful for SFM visualization |
| `UIViewRepresentable` → `MTKView` bridge | Reuse entire pattern |
| Touch input → obstacle system | Extend for goal placement |
| LFO (Low Frequency Oscillator) system | Optional — parameter modulation over time |

### Needs Modification 🟡

| Component | What to Change |
|---|---|
| Particle struct (`Particle`) | Add: `goal_pos` (SIMD2<Float>), `desired_speed` (Float), `radius` (Float) |
| Obstacle system | Replace point circles with line segment walls + normals |
| Config struct (`BoidsConfig`) | Replace with `SFMConfig`: `A`, `B`, `tau`, `desired_speed`, `wall_A`, `wall_B` |

### Needs New Code 🔴

| Component | Description | Estimated Lines |
|---|---|---|
| `simulateSFM` kernel | Core SFM physics: goal force + pedestrian repulsion + wall repulsion | ~120 |
| Wall obstacle buffer | Line segment storage + distance computation | ~40 |
| `SFMViewModel` | Config management, goal placement | ~60 |
| Goal touch handling | Tap-to-set-destination UI | ~30 |
| SFM UI controls | Sliders for A, B, tau, speed, etc. | ~40 |
| Density heatmap *(optional)* | Visualize pedestrian density on a grid | ~50 |
| Flow metrics *(optional)* | Throughput, speed histograms | ~40 |

---

## 5. Proposed Implementation Plan

### Phase 1 — Core Simulation (3-5 hours)

1. **Extend `ShaderTypes.h`** — new `SFMParticle` struct with goal_pos, desired_speed, radius. New `SFMConfig` struct.
2. **Create `SFMShaders.metal`** — `simulateSFM` kernel implementing all Helbing forces.
3. **Create `SFMView.swift`** — SwiftUI → MTKView bridge, based on `BoidsView.swift`.
4. **Create `SFMViewModel.swift`** — config structs, goal management.
5. **Update pipeline states** — register new compute shader.

### Phase 2 — Obstacles & Walls (2-3 hours)

6. **Wall data structure** — array of line segments with normals.
7. **GPU wall buffer** — pass to compute shader.
8. **Touch-based wall drawing** — swipe to create walls.

### Phase 3 — Polish (2-4 hours)

9. **Goal visualization** — arrows, destination markers.
10. **Trail rendering** — reuse blur pipeline from Boids.
11. **Metrics overlay** — density, speed, flow.
12. **Parameter sliders** — real-time SFM tuning.

---

## 6. Risks & Considerations

### GPU-Specific
- **Branch divergence**: The O(n²) loop means all threads run the full loop — fine for Metal's SIMT architecture.
- **Memory bandwidth**: Reading all other particle positions per thread is the main bottleneck. For 10k pedestrians, that's ~80KB read per thread per frame. Metal's texture cache + unified memory on Apple Silicon handles this well.
- **Thread count**: A11+ GPU supports large threadgroups. 512 threads per group with ~20 groups = 10,240 pedestrians per frame at 30fps. Should be comfortable on M-series GPUs.

### Physics-Specific
- **Numerical stability**: Standard SFM can produce oscillations. Need velocity damping (already in Boids `limit_magnitude`). Consider adding a max acceleration cap.
- **Grid vs. direct**: For >5000 pedestrians, spatial indexing (grid) would help, but n² direct computation on GPU is surprisingly efficient thanks to Apple's unified memory architecture.
- **Wall proximity**: Distance-to-line-segment computations on GPU are trivial — no spatial queries needed.

### Architecture-Specific
- **Read-write textures**: Required for trail/blur system. Already handled by existing code (A11+ requirement).
- **CPU-GPU sync**: Frequent readback kills performance. Avoid `extractParticles()` in production — only use for debug overlays.

---

## 7. Verdict: 🟢 HIGHLY FEASIBLE

**Summary:**
This is an excellent base for an SFM demonstration. The Metal compute pipeline is production-quality and maps almost one-to-one to the Social Force Model's computational pattern.

**What to write from scratch:** ~300 lines (kernel, walls, UI)
**What to reuse as-is:** ~2000+ lines (entire Metal pipeline, rendering, buffer management)

### Key Advantages of This Base
1. **Battle-tested GPU pipeline** — already proven across 4 different simulations
2. **Unified memory** — M-series Macs handle GPU particle systems extremely efficiently
3. **Config system** — real-time parameter tuning is trivial to add
4. **Existing UI pattern** — SwiftUI + MTKView bridge is clean and extensible

### Quickstart
To begin: fork/copy `BoidsView.swift` + `BoidsViewModel.swift` + `BoidShaders.metal` + `ShaderTypes.h` as the template, then replace the simulation kernel with SFM physics.
