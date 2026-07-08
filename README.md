# Contribution #1: Add `pointcloud3d` WebGL Pane

**Contribution Number:** 1
**Student:** Afrin Hossain
**Issue:** [fossasia/visdom#686 — Add 3D Point Cloud Visualization](https://github.com/fossasia/visdom/issues/686)
**Status:** Phase III — Conflict Resolution & 4-PR Split — Complete, PRs pending review

---

## Why I Chose This Issue

This issue asks for a first-class 3D point cloud visualization pane in Visdom, with interactive WebGL-based controls for exploring 3D data. Point cloud visualization is central to computer vision research — 3D reconstruction, autonomous driving, robotics, and PointNet-style deep learning models all produce and consume `(N, 3)` XYZ arrays that have no meaningful home in Visdom today.

I chose this issue because it aligns directly with my interest in Computer Vision research, requires no specialized hardware beyond my MacBook, and involves meaningful work across the full stack: Python API design, typed array encoding, React component lifecycle, and WebGL rendering via Three.js.

---

## Understanding the Issue

### Problem Description

Visdom has no native way to render 3D point clouds. The only remotely similar primitive is `vis.scatter`, but it is strictly 2D — it drops the Z dimension entirely. Researchers who generate 3D data (e.g., from a depth sensor, a PointNet model, or a LiDAR scan) have no way to visualize it in Visdom without reaching for an external tool.

### Expected Behavior

`vis.pointcloud3d(xyz, rgb=None, opts=None)` should accept an `(N, 3)` array of XYZ coordinates (and an optional `(N, 3)` RGB color array), and render an interactive 3D pane in the browser with a WebGL orbit camera — rotate, zoom, pan, and reset.

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
- `py/visdom/server/handlers/web_handlers.py` — `UpdateHandler` (window update/patch logic needs to recognize the new type)
- `py/visdom/utils/server_utils.py` — `window()` / `update_window()` (window creation needs to recognize the new type)
- `py/visdom/server/handlers/base_handlers.py` — WebSocket `max_message_size` (needs raising for large point cloud payloads)

*(Corrected July 7, 2026: the original version of this document listed a single `visdom/server.py`; the actual server-side surface area turned out to be three separate handler/util files, identified precisely during the conflict-resolution pass described below.)*

---

## Reproduction Process

### Environment Setup

Set up a local Python/Node environment on macOS. Python 3.9.6 and Node 24.15.0 were already available; no containers needed.

```bash
git clone https://github.com/fossasia/visdom.git
cd visdom
pip install -e .
pip install -r test-requirements.txt   # numpy, matplotlib, torch (CPU)
npm install
python -m visdom.server                # confirms server at localhost:8097
```

Total setup time was around 20 minutes. The only friction was `test-requirements.txt` pulling a CPU-only PyTorch build — the `--extra-index-url` flag for the PyTorch CPU wheel index must be present (it is already included in the file).

**Note (July 7, 2026):** `dev` has since raised its Python floor to 3.12 (`if sys.version_info < (3, 12): raise RuntimeError(...)`, added upstream while this branch sat unreviewed). A fresh clone of current `dev` today needs Python ≥3.12, not 3.9.6, before any of the steps below will work.

Confirmed clean baseline:

```bash
PYTHONPATH=py python3 -m pytest test/ -q
```

**Working branch:** https://github.com/AfrinHossai/visdom/tree/feat/issue-686-pointcloud3d

Branch created from `dev`:

```bash
git checkout dev && git pull origin dev
git checkout -b feat/pointcloud3d-pane
```

**Note (July 7, 2026):** this branch was later merged forward against a much-advanced `dev` and split into 4 separate PR branches — see "Pull Request" below for the current state.

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
- **My findings:** The gap is total — no Python method, no JS component, no server handler, no registry entry. This is a net-new feature with no partial implementation to build on. The closest touchpoints are `NetworkPane.js` (React lifecycle pattern) and the existing `_send(..., endpoint="events")` path (reusable JSON transport).

---

---

## Solution Approach

### Analysis

The root cause is absence, not a bug. Nothing for `pointcloud3d` exists anywhere in the codebase. The key design question is *transport*: how do large typed arrays of XYZ floats and RGB bytes travel from Python to the browser? Three options exist — inline base64, a new binary HTTP endpoint, or WebSocket streaming. Inline base64 was chosen for MVP because it reuses the existing `_send` JSON path with zero server changes and keeps the payload self-contained.

### Proposed Solution

Python validates and normalises the input arrays, optionally downsamples them, encodes them as inline base64 inside a versioned JSON envelope, and ships via `_send`. The browser decodes the base64 back to typed arrays, builds a `THREE.BufferGeometry + THREE.Points` mesh, and hands it to a custom `OrbitController` for interaction.

### Implementation Plan

Using the UMPIRE framework:

**Understand:** Visdom lacks any 3D point cloud primitive. `vis.scatter` is 2D-only. Researchers using LiDAR, PointNet, or depth sensors have no visualization path.

**Match:** `NetworkPane` and `EmbeddingsPane` in `js/panes/` provide the React class component pattern. `js/settings.js` `PANES` map is the single registration point. `_send({...}, endpoint="events")` already carries arbitrary typed payloads — no new server routes needed.

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
| `_pc3d_validate_opts(opts)` | Validate markersize, opacity, max_points, downsample |
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

**Implement:** https://github.com/AfrinHossai/visdom/tree/feat/pointcloud3d-pane

**Review:** Self-review against `CONTRIBUTING.md` commit message conventions (conventional commits: `feat:`, `fix:`, `test:`, `build:`). Confirm no existing pane types are affected by the `settings.js` change. Run the full test suite before opening the PR.

**Evaluate:** Re-running the reproduction steps should show:
- `viz.pointcloud3d(xyz)` returns a window ID with no exception
- A WebGL canvas pane appears in the browser
- All orbit interactions work (rotate, zoom, pan, reset)
- `PYTHONPATH=py python3 -m pytest test/test_pointcloud3d.py -q` → 189 passed, 3 skipped
- `grep -c "PointCloudPane\|OrbitController" static/js/main.js` → non-zero

---

## Implementation Summary

**Branch:** https://github.com/AfrinHossai/visdom/tree/feat/pointcloud3d-pane
**Total commits:** 23 | **Files changed:** 7

### What was built

- `vis.pointcloud3d(xyz, rgb=None, win=None, env=None, opts=None)` — full Python method with input validation, optional downsampling, and inline base64 transport
- 6 module-level helper functions in `py/visdom/__init__.py` handling the full validate → downsample → encode pipeline
- `js/panes/PointCloudPane.js` — new React class component with:
  - Custom `OrbitController` (rotate/zoom/pan/reset) — no Three.js upgrade needed
  - `THREE.BufferGeometry + THREE.Points` rendering with per-point RGB support
  - Dirty render loop using `requestAnimationFrame` (only fires on actual change)
  - Proper GPU teardown (`dispose()` on geometry, material, renderer) on unmount
- `js/settings.js` updated to register `pointcloud3d` pane type
- Server-side `window()` handler patched to recognise the new type
- 2 runnable demo functions in `example/components/plot_pointcloud.py`

### Update (July 7, 2026): Conflict Resolution & 4-PR Split

`dev` advanced roughly 40 commits while this branch sat unreviewed — Confusion Matrix, Sankey, and Table panes were added by other contributors, the Python floor was raised to 3.12, and a batch of unrelated fixes and CI changes landed. As a result, the PR opened from this branch (fossasia/visdom#1545) started showing merge conflicts on GitHub, and a review bot flagged its diff as exceeding a 150,000-character review limit.

Resolution work, in order:

1. **Committed pending doc/version work** that was sitting unstaged locally (README API entry + section, `website/docs/api/point-cloud.md`, sidebar wiring) — matched the PR's own checklist but had never actually been committed.
2. **Merged `upstream/dev`.** Real conflicts were limited to four files: `example/demo.py` and `py/visdom/__init__.py` (both cases were two branches inserting new content — our `pointcloud3d()`/import and dev's new `sankey()`/imports — at the identical location; resolved by taking the union of both sides), plus `py/visdom/static/js/main.js` and `.map`. The static JS files are build artifacts; per this repo's own `AGENTS.md` ("never manually edit compiled files in `py/visdom/static/`"), these were taken wholesale from upstream rather than hand-merged — the frontend source isn't in any built bundle yet either way, and the next automated "update static/js files" bot commit picks it up once the frontend PR merges.
3. **Found and fixed two latent issues** while doing this, unrelated to the merge itself: a dead `_float_img_to_uint8()` helper (defined in the original implementation commit, never called anywhere — `_pc3d_validate_rgb` has its own separate inline conversion logic) was removed; and a missing `py/visdom/__init__.pyi` type stub for `pointcloud3d()` — required by this repo's own PR checklist for interface changes, but never added in the original implementation — was written in, matching the existing stub style.
4. **Re-scoped the single 3,300+ line PR into 4 independently reviewable PRs**, each on its own branch off current `dev`. Verified the split is lossless and non-overlapping via a scratch octopus-merge of all 4 branches together (zero conflicts — confirms the four file-sets are fully disjoint) and a diff of that merged result against the fully-resolved source tree (byte-identical except for two intentional omissions: a 601-line internal AI-planning doc under `docs/superpowers/specs/`, not shippable repo content; and, initially, the `VERSION` bump — see point 6 below).
5. **Split `test/test_pointcloud3d.py`** along its own existing class boundaries into two files: the original file keeps everything except `TestServerWindowHandler` (189 tests, ships with PR 1), and a new `test/test_pointcloud3d_server.py` carries just that one class (6 tests, ships with PR 2). Confirmed the split has zero code dependency in either direction — the server-side tests hand-build fixture dicts and call `server_utils.window()`/`update_window()` directly, never `vis.pointcloud3d()` — so the two PRs are independently mergeable in either order.
6. **Bumped `py/visdom/VERSION` from `0.2.4` to `0.3.0` identically in all 4 PRs**, so every PR's checklist is fully checkable. This repo's own history (`git log -- py/visdom/VERSION`) shows version bumps are normally a separate maintainer release PR to `master`, not part of feature PRs — bumping it here across all 4 is a deliberate exception, made safe by using the exact same target value in every branch (verified via another scratch merge: identical changes to the same line don't conflict, regardless of merge order).
7. **Adopted a `[visdom-686]` commit-title prefix** across all 4 branches — a new convention introduced this round, not a pre-existing repo pattern.
8. **Left the original PR #1545 and its branch untouched on GitHub** — the 4 new PRs are the actual review path; #1545 gets closed manually once they're in.

All 4 branches were pushed to the student's fork (`AfrinHossai/visdom`) and verified with `black` (v23.1.0, CI-pinned) and the full test suite before and after every change.

### Key commits

| Commit | Description |
|---|---|
| `da4856b` | Design spec |
| `65fad03` | `_pc3d_validate_xyz` + tests |
| `2e3a4b1` | Full Python `pointcloud3d` method |
| `0533e19` | `OrbitController` class |
| `2da493a` | `buildGeometry`, `disposeCloud`, `disposeAll` |
| `c83cac2` | Server handler fix + server-side tests |
| `6069f06` | JS bundle rebuild |
| `464efa4` | Final test expansion (79 → 192 tests) |

### Split commits (July 7, 2026)

| Commit | Branch | Description |
|---|---|---|
| `e8731a6` | `feat/pointcloud3d-1-client-api` | `[visdom-686]` Python client API, 189 tests, `.pyi` stub, VERSION bump |
| `917a211` | `feat/pointcloud3d-2-server-handlers` | `[visdom-686]` server-side wiring, 6 tests, VERSION bump |
| `1c0d8cb` | `feat/pointcloud3d-3-frontend-pane` | `[visdom-686]` `PointCloudPane` component, VERSION bump |
| `aa96f3d` | `feat/pointcloud3d-4-examples-docs` | `[visdom-686]` examples, Cypress test, docs, VERSION bump |

Across the 4 branches: 17 distinct files touched (confirmed via the scratch octopus-merge), 195 tests total, 0 conflicts merging the 4 together.

### Bugs caught during implementation

- `_pc3d_encode_array` was not enforcing the target dtype before `.tobytes()` — a float64 input would silently produce a double-width payload. Fixed by casting before encoding.
- Server `window()` handler did not recognise `"pointcloud3d"`, causing the payload to be discarded silently on page refresh. Found only during end-to-end testing.
- *(July 7, 2026)* A `_float_img_to_uint8()` helper was defined but never called anywhere in the codebase — dead code left over from an earlier, later-abandoned approach to RGB conversion. Found via a full-codebase grep during the conflict-resolution pass, and removed.
- *(July 7, 2026)* The original implementation never added a `py/visdom/__init__.pyi` type stub for `pointcloud3d()`, despite this being an explicit item in the repo's own PR checklist. Found by diffing the `.pyi` file against `dev` and noticing it was untouched; added to match the existing stub style.

---

## Testing Strategy

### Unit Tests

- [x] `TestXyzValidation` — shape, dtype coercion, NaN/Inf, empty, non-numeric inputs
- [x] `TestRgbValidation` — uint8 passthrough, integer range check, float [0,1] scaling, float [0,255], NaN, wrong shape
- [x] `TestOptsValidation` — markersize ≤ 0, opacity out of range, max_points ≤ 0, conflicting downsample='none', invalid mode string
- [x] `TestDownsampling` — stride count, random count, reproducibility with seed, no-op when N ≤ max_points, 200k warning
- [x] `TestPayload` — version/transport fields, xyz dtype/shape/encoding, rgb presence/absence, num_points fields, bounds fields, base64 round-trip
- [x] `TestBoundsHelper` — unit cube center and radius
- [x] `TestIntegrationFullPipeline` — end-to-end `vis.pointcloud3d()` calls with `Visdom._send` mocked
- [x] `TestServerWindowHandler` — *(as of July 7, 2026, lives in its own file `test/test_pointcloud3d_server.py`, shipped in PR 2 instead of PR 1)* — `window()`/`update_window()` routing, opts nesting, no-crash-on-empty-update, opts preserved across partial updates

### Integration Tests

- [x] Cypress smoke test — pane renders with `<canvas>` after demo call
- [x] Cypress interaction test — left-drag and scroll do not crash the pane
- [x] Server-side `window()` handler recognises `pointcloud3d` type

### Manual Testing

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

Confirmed: `OK` printed; pane appeared at `http://localhost:8097` with all orbit controls working correctly.

### Testing Notes
```
PYTHONPATH=py python3 -m pytest test/test_pointcloud3d.py -v
```

| Result | Count |
|---|---|
| Passed | 189 |
| Skipped | 3 (PyTorch-dependent paths — skipped when `torch` unavailable) |
| Failed | 0 |

The 3 skipped tests cover `@pytorch_wrap` tensor coercion paths. They pass in environments where PyTorch is installed and are skipped cleanly otherwise — no false negatives.

**Update (July 7, 2026):** re-ran the full suite against the merged, conflict-resolved tree, and again independently after splitting into `test/test_pointcloud3d.py` + `test/test_pointcloud3d_server.py`. Both runs: **195 passed, 0 skipped, 0 failed** (PyTorch was available in this environment, so none of the `@pytorch_wrap` paths were skipped this time). The 195 total splits as 189 (PR 1) + 6 (PR 2), matching the pre-split single-file count exactly — confirmed no test was lost or duplicated in the split.

Test growth over the implementation cycle:

| Milestone | Test count |
|---|---|
| Initial helpers (xyz + rgb) | 20 |
| Opts + downsample helpers | 36 |
| Full payload + bounds | 79 |
| Exhaustive gap analysis pass | 142 |
| Final expansion | 192 |
| Post-merge, pre-split (July 7, 2026) | 195 |
| Split across 2 files, PR 1 + PR 2 (July 7, 2026) | 189 + 6 = 195 |

---

## Implementation Notes

### Week 1 (June 8) Progress

Entire feature designed, implemented, bug-fixed, and test-hardened in a single focused session. Key decisions and challenges:

- **Transport choice:** Evaluated binary HTTP endpoint vs. inline base64. Chose base64 to avoid any server changes — the existing `_send` path handles it transparently.
- **OrbitController:** Decided against upgrading Three.js (v0.105.2 is pinned) and wrote a lightweight custom controller instead. Avoids breaking existing panes.
- **Bounds precomputation:** Initially the JS side computed `geometry.computeBoundingSphere()` client-side. Moved to server-side precomputation so the JS never iterates large typed arrays during camera setup — measurable perf win for 100k+ point clouds.

### Week 2 (July 7) Progress — Conflict Resolution & PR Split

The branch had gone stale after roughly a month of sitting unreviewed while `dev` kept moving. Key decisions and challenges this round:

- **Diagnosing the conflicts before touching anything:** rather than immediately running `git merge` and fighting whatever came up, first used `git merge-tree --write-tree` to see exactly which files would conflict and why, before making any changes. This turned up only 4 real conflicts out of 37 files `dev` had touched — the rest auto-merged cleanly.
- **Recognizing build artifacts as a special case:** `py/visdom/static/js/main.js` and `.map` showed up as conflicts, but per this repo's own `AGENTS.md`, those files should never be hand-edited — they're CI-generated. Rather than trying to reconcile a diff in a minified bundle, the fix was to just take upstream's version wholesale and let the next automated build pick up the new frontend source later.
- **Treating "split into smaller PRs" as an architecture question, not a mechanical one:** rather than arbitrarily cutting the diff at line counts, used this repo's own `.agents/skills/adding-pane/` documentation (client → server → frontend → registration → demo → tests) as the layering, and then specifically checked whether the layers had any *code* dependency on each other (not just file-overlap) before deciding they could be independent, order-agnostic PRs rather than a rigid dependency chain.
- **Verifying a split quantitatively, not just by inspection:** used a disposable scratch branch to octopus-merge all 4 split branches together and diff the result against the original resolved tree. This caught the split's correctness (or would have caught a mistake) far more reliably than manually re-reading each diff.
- **Same-value changes across sibling branches don't conflict:** when asked to bump `VERSION` in all 4 PRs despite the conflict risk, realized that if every branch bumps to the *exact same* target value, git's 3-way merge treats that as a non-conflicting identical change — verified this before committing to the approach, rather than assuming it would create the very problem being warned about.

---

## Pull Request

**Original PR:** [fossasia/visdom#1545](https://github.com/fossasia/visdom/pull/1545) — left open and untouched on GitHub; superseded by the 4 PRs below. Will be closed by the student once those merge.

**Status (July 7, 2026):** #1545 went stale against `dev` (merge conflicts) and was flagged by a review bot for exceeding a 150,000-character diff limit. Resolved and re-scoped into 4 independently reviewable PRs, each pushed to `AfrinHossai/visdom` and ready to open via the compare links below (not yet formally opened as PR objects on `fossasia/visdom` as of this writing).

### PR 1 of 4 — Python client API

**Branch:** `feat/pointcloud3d-1-client-api`
**Compare link:** https://github.com/fossasia/visdom/compare/dev...AfrinHossai:visdom:feat/pointcloud3d-1-client-api?expand=1
**Title:** `[visdom-686] feat(pointcloud3d): add vis.pointcloud3d() Python client API`

Adds `vis.pointcloud3d(xyz, rgb=None, win=None, env=None, opts=None)` — validation, normalization, optional downsampling (hard cap 500,000 points), and versioned base64 payload encoding. Includes the `py/visdom/__init__.pyi` type stub and 189 unit tests. Self-contained; no dependency on the other 3 PRs. Checklist: all 4 boxes checked (VERSION bump `0.2.4` → `0.3.0` included by deliberate exception — see "Update (July 7, 2026)" above; documentation itself lands in PR 4).

### PR 2 of 4 — Server-side wiring

**Branch:** `feat/pointcloud3d-2-server-handlers`
**Compare link:** https://github.com/fossasia/visdom/compare/dev...AfrinHossai:visdom:feat/pointcloud3d-2-server-handlers?expand=1
**Title:** `[visdom-686] feat(pointcloud3d): wire pointcloud3d pane type into the server`

Extends `UpdateHandler`/`server_utils` to recognize `pointcloud3d`: flat content routing instead of the Plotly data/layout shape, opts nested under `p["opts"]`, a manual JSON-patch builder (`_update_packet_pc3d`) that avoids blocking the IOLoop on multi-MB payloads and only bumps `contentID` when point data actually changes, and a 64 MiB websocket message-size cap. 6 unit tests in a new `test/test_pointcloud3d_server.py`. No dependency on PR 1 — the tests hand-build fixtures rather than calling `vis.pointcloud3d()`. Checklist: VERSION and code-style checked; documentation checkbox intentionally unchecked (internal server wiring has no separate user-facing docs entry of its own).

### PR 3 of 4 — Frontend `PointCloudPane` component

**Branch:** `feat/pointcloud3d-3-frontend-pane`
**Compare link:** https://github.com/fossasia/visdom/compare/dev...AfrinHossai:visdom:feat/pointcloud3d-3-frontend-pane?expand=1
**Title:** `[visdom-686] feat(pointcloud3d): add PointCloudPane WebGL frontend component`

New `PointCloudPane` React component (`THREE.BufferGeometry` + `THREE.Points`, dirty render loop, custom `OrbitController`, WebGL context-loss recovery, reactive opts updates without full geometry rebuilds) plus `js/settings.js` registration. No JS unit-test layer exists in this repo (Cypress only, in PR 4); style was verified by direct comparison against `TextPane.js`'s conventions rather than a linter run, since no Node toolchain was available in the preparation environment. Checklist: all 4 boxes checked.

### PR 4 of 4 — Examples, Cypress test, docs (merge last)

**Branch:** `feat/pointcloud3d-4-examples-docs`
**Compare link:** https://github.com/fossasia/visdom/compare/dev...AfrinHossai:visdom:feat/pointcloud3d-4-examples-docs?expand=1
**Title:** `[visdom-686] docs(pointcloud3d): examples, Cypress test, and docs for pointcloud3d`

Demo functions (`plot_pointcloud_basic`, `plot_pointcloud_rgb`), the Cypress end-to-end spec, the README API section, and the new `website/docs/api/point-cloud.md` page. This is the only PR in the stack with a real ordering dependency: its diff is independent of PRs 1–3 (disjoint files, reviewable now), but the Cypress spec needs all three merged to actually go green in CI. Checklist: all 4 boxes checked — this is the one PR where "documentation updated" is literally true.

**Maintainer Feedback:** *(awaiting review — 4 PRs pending)*

**Status:** Ready for review

---

## Learnings & Reflections

### Technical Skills Gained

- Typed array encoding: little-endian float32/uint8 layout, base64 overhead, and how JavaScript `Float32Array` reconstructs bytes from a `Uint8Array` buffer
- Three.js BufferGeometry API: `setAttribute` vs. `addAttribute` (version compatibility), `VertexColors` vs `NoColors`, `sizeAttenuation`, `depthWrite` for transparent materials
- React class component teardown: importance of `dispose()` calls on GPU resources to avoid memory leaks (`WebGLRenderer`, `BufferGeometry`, `Material`)
- Dirty render loops: `requestAnimationFrame` should only fire when state actually changes
- *(July 7, 2026)* Diagnosing merge conflicts before resolving them, using `git merge-tree` to see the actual conflict surface instead of discovering it hunk-by-hunk mid-merge
- *(July 7, 2026)* Verifying a large-PR split is lossless and conflict-free quantitatively (scratch octopus-merge + diff against source) instead of trusting manual inspection alone
- *(July 7, 2026)* Recognizing that "both branches changed the same line to the same value" is not a git conflict — useful for coordinating an identical change (like a version bump) across otherwise-independent sibling branches

### Challenges Overcome

- **dtype enforcement bug:** Silent float64 → double-width payload. Found by writing a base64 round-trip test that decoded the payload and compared byte counts.
- **Camera fit:** Finding the right `DIST_FACTOR` for the initial orbit radius across wildly different point cloud scales required several iterations.
- **Server handler gap:** Not visible from the JS side alone — found only by testing end-to-end and noticing panes disappeared on page refresh.
- *(July 7, 2026)* **A PR going stale from sitting too long:** roughly 40 upstream commits landed while this branch waited for review, turning a clean PR into a conflicting, oversized one. The fix wasn't just resolving conflicts — it required rethinking how the change should be delivered at all.
- *(July 7, 2026)* **A review bot's diff-size limit forcing an architectural decision:** rather than just shrinking the diff cosmetically, used it as the occasion to actually split the change along its natural seams (client / server / frontend / integration), which produced a better-organized contribution than the original single PR.

### What I'd Do Differently Next Time

Write the end-to-end smoke test earlier — before all unit tests are green. The server handler bug would have been caught immediately instead of at the final integration step.

*(July 7, 2026)* Also: open the PR in smaller pieces from the very start, rather than building the entire stack on one branch and only splitting it after it went stale. The 4-PR structure used to rescue this contribution is basically what the first submission should have looked like.

---

## Resources Used

- [Three.js BufferGeometry documentation](https://threejs.org/docs/#api/en/core/BufferGeometry)
- [Three.js PointsMaterial documentation](https://threejs.org/docs/#api/en/materials/PointsMaterial)
- [fossasia/visdom CONTRIBUTING.md](https://github.com/fossasia/visdom/blob/dev/CONTRIBUTING.md)
- [fossasia/visdom#686 — original issue thread](https://github.com/fossasia/visdom/issues/686)
- [MDN — Typed Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays)
- Existing `NetworkPane.js` and `EmbeddingsPane.js` in this repo as React lifecycle references
- *(July 7, 2026)* This repo's own `AGENTS.md` and `.agents/skills/adding-pane/`, `.agents/skills/release-process/` — used to ground the PR split's layering and the VERSION-bump question in the project's actual documented conventions rather than guesswork
