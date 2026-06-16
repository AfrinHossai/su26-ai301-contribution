# Contribution #1: Add `pointcloud3d` WebGL Pane

**Contribution Number:** 1  
**Student:** Afrin Hossain  
**Issue:** [fossasia/visdom#686 — Add 3D Point Cloud Visualization](https://github.com/fossasia/visdom/issues/686)  
**Status:** Phase II — Complete

---

## Why I Chose This Issue

This issue asks for a first-class 3D point cloud visualization pane in Visdom, with
interactive WebGL-based controls for exploring 3D data. Point cloud visualization
is central to computer vision research — 3D reconstruction, autonomous driving, robotics,
and PointNet-style deep learning models all produce and consume `(N, 3)` XYZ arrays that
have no meaningful home in Visdom today.

I chose this issue because it aligns directly with my interest in Computer Vision research,
requires no specialized hardware beyond my MacBook, and involves meaningful work across
the full stack: Python API design, typed array encoding, React component lifecycle, and
WebGL rendering via Three.js.

---

## Understanding the Issue

### Problem Description

Visdom has no native way to render 3D point clouds. The only remotely similar primitive
is `vis.scatter`, but it is strictly 2D — it drops the Z dimension entirely. Researchers
who generate 3D data (e.g., from a depth sensor, a PointNet model, or a LiDAR scan)
have no way to visualize it in Visdom without reaching for an external tool.

### Expected Behavior

`vis.pointcloud3d(xyz, rgb=None, opts=None)` should accept an `(N, 3)` array of XYZ
coordinates (and an optional `(N, 3)` RGB color array), and render an interactive 3D
pane in the browser with a WebGL orbit camera — rotate, zoom, pan, and reset.

### Current Behavior

```python
viz.pointcloud3d(xyz)
# → AttributeError: 'Visdom' object has no attribute 'pointcloud3d'
```

On the JS side, the pane type is completely absent from the registry:

```bash
grep -n "pointcloud3d" js/settings.js
# → (no output)
```

### Affected Components

- `py/visdom/__init__.py` — Python API and validation layer
- `js/panes/PointCloudPane.js` — new WebGL React component (does not exist yet)
- `js/settings.js` — central pane type registry
- `visdom/server.py` — server-side `window()` handler (needs to recognise the new type)

---

## Reproduction Process

### Environment Setup

Set up a local Python/Node environment on macOS. Python 3.9.6 and Node 24.15.0 were
already available; no containers needed.

```bash
git clone https://github.com/fossasia/visdom.git
cd visdom
pip install -e .
pip install -r test-requirements.txt   # numpy, matplotlib, torch (CPU)
npm install
python -m visdom.server                # confirms server at localhost:8097
```

Total setup time was around 20 minutes. The only friction was `test-requirements.txt`
pulling a CPU-only PyTorch build — the `--extra-index-url` flag for the PyTorch CPU
wheel index must be present (it is already included in the file).

Confirmed clean baseline:

```bash
PYTHONPATH=py python3 -m pytest test/ -q
```

**Working branch:** https://github.com/pahmed/visdom/tree/feat/pointcloud3d-pane

Branch created from `dev`:

```bash
git checkout dev && git pull origin dev
git checkout -b feat/pointcloud3d-pane
```

### Steps to Reproduce

1. Start the Visdom server: `python -m visdom.server`
2. In a Python shell:

```python
import numpy as np
from visdom import Visdom

viz = Visdom()
xyz = np.random.randn(50_000, 3).astype('float32')
viz.pointcloud3d(xyz)
```

3. **Expected:** 3D point cloud pane appears at `http://localhost:8097`
4. **Actual:** `AttributeError: 'Visdom' object has no attribute 'pointcloud3d'`

### Reproduction Evidence

- **Commit showing reproduction:** `da4856b` (design spec added before any implementation)
- **My findings:** The gap is total — no Python method, no JS component, no server handler,
  no registry entry. This is a net-new feature with no partial implementation to build on.
  The closest touchpoints are `NetworkPane.js` (React lifecycle pattern) and the existing
  `_send(..., endpoint="events")` path (reusable JSON transport).

---

## Solution Approach

### Analysis

The root cause is absence, not a bug. Nothing for `pointcloud3d` exists anywhere in the
codebase. The key design question is *transport*: how do large typed arrays of XYZ floats
and RGB bytes travel from Python to the browser? Three options exist — inline base64,
a new binary HTTP endpoint, or WebSocket streaming. Inline base64 was chosen for MVP
because it reuses the existing `_send` JSON path with zero server changes and keeps
the payload self-contained.

### Proposed Solution

Python validates and normalises the input arrays, optionally downsamples them, encodes
them as inline base64 inside a versioned JSON envelope, and ships via `_send`. The browser
decodes the base64 back to typed arrays, builds a `THREE.BufferGeometry + THREE.Points`
mesh, and hands it to a custom `OrbitController` for interaction.

### Implementation Plan

Using the UMPIRE framework:

**Understand:** Visdom lacks any 3D point cloud primitive. `vis.scatter` is 2D-only.
Researchers using LiDAR, PointNet, or depth sensors have no visualization path.

**Match:** `NetworkPane` and `EmbeddingsPane` in `js/panes/` provide the React class
component pattern. `js/settings.js` `PANES` map is the single registration point.
`_send({...}, endpoint="events")` already carries arbitrary typed payloads — no new
server routes needed.

**Plan:**

*Architecture — end-to-end flow:*

```
vis.pointcloud3d(xyz, rgb, opts)
  → validate + normalize (xyz → float32 LE, rgb → uint8)
  → optional Python-side downsample (stride or random)
  → base64-encode typed arrays + precompute bounds
  → _send({data:[{type:"pointcloud3d", content:{...}}]}, endpoint="events")
  → existing JSON event path (server unchanged)
  → js/settings.js PANES["pointcloud3d"] → PointCloudPane
  → decode base64 → Float32Array / Uint8Array
  → THREE.BufferGeometry + THREE.Points + PointsMaterial
  → custom OrbitController (left-drag rotate, scroll zoom, shift-drag pan, dblclick reset)
  → dirty render loop (requestAnimationFrame only on data/camera/resize change)
```

*Files changed:*

| File | Action | Responsibility |
|---|---|---|
| `py/visdom/__init__.py` | Modify | 6 module-level helpers + `pointcloud3d` class method |
| `js/panes/PointCloudPane.js` | Create | `OrbitController` + `PointCloudPane` React component |
| `js/settings.js` | Modify | Register pane type + default size `[30, 24]` |
| `example/components/plot_pointcloud.py` | Create | 2 demo functions (basic + RGB) |
| `example/demo.py` | Modify | Import + call demo functions |
| `test/test_pointcloud3d.py` | Create | 192 Python unit tests |
| `cypress/integration/pointcloud3d.js` | Create | Cypress smoke + interaction test |

*Python helpers (in call order):*

| Helper | Purpose |
|---|---|
| `_pc3d_validate_xyz(xyz)` | Coerce to float32 (N, 3), reject NaN/Inf/wrong shape |
| `_pc3d_validate_rgb(rgb, N)` | Coerce to uint8 (N, 3); auto-scale float [0, 1] → [0, 255] |
| `_pc3d_validate_opts(opts)` | Validate markersize, opacity, max\_points, downsample |
| `_pc3d_downsample(xyz, rgb, opts)` | Stride or random subsample; warns if N > 200k |
| `_pc3d_compute_bounds(xyz)` | Precomputes {min, max, center, radius} server-side |
| `_pc3d_encode_array(arr, dtype)` | Base64-encodes typed array with dtype/shape envelope |

*Payload shape:*

```json
{
  "version": 1,
  "transport": "inline_base64",
  "xyz": { "dtype": "float32", "shape": [N, 3], "encoding": "base64", "byte_order": "little", "data": "..." },
  "rgb": { "dtype": "uint8",   "shape": [N, 3], "encoding": "base64", "byte_order": "little", "data": "..." },
  "num_points_original": N,
  "num_points_rendered": N,
  "bounds": { "min": [...], "max": [...], "center": [...], "radius": 1.23 }
}
```

`rgb` key omitted when no colors provided.

*JS component lifecycle:*

```
componentDidMount    → initScene → initController → buildGeometry → requestRender
componentDidUpdate  → disposeCloud (if content changed) → buildGeometry → requestRender
componentWillUnmount → disposeAll (WebGLRenderer + geometry + material + controller)
```

*Camera / orbit controls:*

| Interaction | Action |
|---|---|
| Left-drag | Rotate (spherical theta/phi) |
| Scroll wheel | Zoom (exponential radius scale) |
| Shift+drag or right-drag | Pan (translate target in camera-local XY) |
| Double-click | Reset to saved initial state |
| Toolbar reset button | Same as double-click |

**Implement:** https://github.com/pahmed/visdom/tree/feat/pointcloud3d-pane

**Review:** Self-review against `CONTRIBUTING.md` commit message conventions
(conventional commits: `feat:`, `fix:`, `test:`, `build:`). Confirm no existing
pane types are affected by the `settings.js` change. Run the full test suite
before opening the PR.

**Evaluate:** Re-running the reproduction steps should show:
- `viz.pointcloud3d(xyz)` returns a window ID with no exception
- A WebGL canvas pane appears in the browser
- All orbit interactions work (rotate, zoom, pan, reset)
- `PYTHONPATH=py python3 -m pytest test/test_pointcloud3d.py -q` → 189 passed, 3 skipped
- `grep -c "PointCloudPane\|OrbitController" static/js/main.js` → non-zero

---

## Testing Strategy

### Unit Tests

- [x] `TestXyzValidation` — shape, dtype coercion, NaN/Inf, empty, non-numeric inputs
- [x] `TestRgbValidation` — uint8 passthrough, integer range check, float [0,1] scaling, float [0,255], NaN, wrong shape
- [x] `TestOptsValidation` — markersize ≤ 0, opacity out of range, max\_points ≤ 0, conflicting downsample='none', invalid mode string
- [x] `TestDownsampling` — stride count, random count, reproducibility with seed, no-op when N ≤ max\_points, 200k warning
- [x] `TestPayload` — version/transport fields, xyz dtype/shape/encoding, rgb presence/absence, num\_points fields, bounds fields, base64 round-trip
- [x] `TestBoundsHelper` — unit cube center and radius

**Total: 189 passed, 3 skipped (PyTorch-dependent paths skipped when torch unavailable)**

### Integration Tests

- [x] Cypress smoke test — pane renders with `<canvas>` after demo call
- [x] Cypress interaction test — left-drag and scroll do not crash the pane
- [x] Server-side `window()` handler recognises `pointcloud3d` type

### Manual Testing

Started Visdom server and ran the basic demo:

```bash
python -m visdom.server &
python -c "
from visdom import Visdom
import numpy as np
viz = Visdom()
xyz = np.random.randn(50_000, 3).astype('float32')
rgb = np.random.randint(0, 256, (50_000, 3), dtype='uint8')
viz.pointcloud3d(xyz, rgb=rgb, opts=dict(title='Smoke test', markersize=2))
print('OK')
"
```

Confirmed: `OK` printed; pane appeared at `http://localhost:8097` with interactive orbit
controls working correctly.

---

## Implementation Notes

### Week 1 (June 8) Progress

Entire feature designed, implemented, bug-fixed, and test-hardened in a single focused
session. Key decisions and challenges:

- **Transport choice:** Evaluated binary HTTP endpoint vs. inline base64. Chose base64
  to avoid any server changes — the existing `_send` path handles it transparently.
- **OrbitController:** Decided against upgrading Three.js (v0.105.2 is pinned) and wrote
  a lightweight custom controller instead. Avoids breaking existing panes that depend on
  the current Three.js version.
- **Bounds precomputation:** Initially the JS side computed `geometry.computeBoundingSphere()`
  client-side. Moved to server-side precomputation so the JS never iterates large typed
  arrays during camera setup — measurable perf win for 100k+ point clouds.
- **Bug found in review:** `_pc3d_encode_array` was not enforcing the target dtype before
  calling `.tobytes()` — a float64 array would silently produce a double-width payload.
  Fixed by casting to the declared dtype before encoding.
- **Server handler gap:** The `window()` server-side handler did not recognise
  `"pointcloud3d"` as a valid pane type, causing it to fall through to the default
  (which discards the payload). Fixed by adding the type to the handler's dispatch list.

### Code Changes

- **Files modified/created:** 7 files (see Files Changed table above)
- **Key commits:**
  - `da4856b` — design spec
  - `65fad03` — `_pc3d_validate_xyz` + tests
  - `2e3a4b1` — full Python `pointcloud3d` method
  - `0533e19` — `OrbitController` class
  - `2da493a` — `buildGeometry`, `disposeCloud`, `disposeAll`
  - `c83cac2` — server handler fix
  - `6069f06` — JS bundle rebuild
  - `464efa4` — final test expansion (79 → 192 tests)
- **Total commits:** 23 on branch `feat/pointcloud3d-pane`

---

## Pull Request

**PR Link:** *(pending — branch ready, PR not yet opened)*

**PR Description draft:**

> Add `vis.pointcloud3d(xyz, rgb=None, win=None, env=None, opts=None)` — a first-class
> 3D point cloud pane backed by Three.js `BufferGeometry + Points` and a custom orbit
> controller. Uses inline base64 transport over the existing `_send` JSON event path;
> no new server routes. Includes 192 Python unit tests, Cypress smoke test, and two
> runnable demo functions.

**Maintainer Feedback:** *(awaiting review)*

**Status:** Ready for review

---

## Learnings & Reflections

### Technical Skills Gained

- Typed array encoding: understanding little-endian float32/uint8 layout, base64 encoding
  overhead, and how JavaScript `Float32Array` reconstructs bytes from a `Uint8Array` buffer
- Three.js BufferGeometry API: `setAttribute` vs. `addAttribute` (version compatibility),
  `VertexColors` vs `NoColors`, `sizeAttenuation`, `depthWrite` for transparent materials
- React class component teardown: importance of `dispose()` calls on GPU resources
  (`WebGLRenderer`, `BufferGeometry`, `Material`) to avoid memory leaks
- Dirty render loops: why `requestAnimationFrame` should only fire when state actually
  changes (camera move, data update, resize) rather than every frame

### Challenges Overcome

- **dtype enforcement bug:** Silent float64 → double-width payload in `_pc3d_encode_array`.
  Found by writing a base64 round-trip test that decoded the payload and compared byte counts.
- **Camera fit:** Finding the right `DIST_FACTOR` for the initial orbit radius so the
  cloud fills most of the viewport across wildly different point cloud scales required
  several iterations.
- **Server handler gap:** Not obvious from reading the JS side alone — only found by
  testing end-to-end and noticing panes disappeared on page refresh.

### What I'd Do Differently Next Time

Write the end-to-end smoke test (start server, call method, check browser) earlier
in the process rather than after all unit tests are green. The server handler bug would
have been caught immediately instead of at the final integration step.

---

## Resources Used

- [Three.js BufferGeometry documentation](https://threejs.org/docs/#api/en/core/BufferGeometry)
- [Three.js PointsMaterial documentation](https://threejs.org/docs/#api/en/materials/PointsMaterial)
- [fossasia/visdom CONTRIBUTING.md](https://github.com/fossasia/visdom/blob/dev/CONTRIBUTING.md)
- [fossasia/visdom#686 — original issue thread](https://github.com/fossasia/visdom/issues/686)
- [MDN — Typed Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays)
- Existing `NetworkPane.js` and `EmbeddingsPane.js` in this repo as React lifecycle references
