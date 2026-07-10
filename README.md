# three-boost-patch

A [bun patch](https://bun.com/docs/install/patch) for **three.js r0.185.0** that reduces CPU cost
in scenes with many animated skinned meshes, and fixes a memory leak in the WebGPU renderer.

Measured on a real-game workload (400–500 animated characters, ~55 bones each, shared node
materials, WebGPU renderer): **−5 to −6 % frame time, −8 to −9 % renderer CPU**, and **−26 %** on
the animation slice (mixer + matrix updates + skeleton flatten) in isolation. three's full unit
suite passes unchanged.

## Usage

Copy `patches/three@0.185.0.patch` into your project's `patches/` folder and register it:

```json
{
  "dependencies": {
    "three": "0.185.0"
  },
  "patchedDependencies": {
    "three@0.185.0": "patches/three@0.185.0.patch"
  }
}
```

Then `bun install`. To confirm the patch is live at runtime, check
`globalThis.__MANATHREE__` in the console — the patch sets it to a build marker string.

The patch edits the built bundles (`build/three.core.js` and `build/three.webgpu.js`), so it
covers every entry point (`three`, `three/webgpu`, `three/tsl`). It is exactly version-locked to
`three@0.185.0`.

## What it changes and why

### Performance

1. **Lazy quaternion→euler sync** (`Euler`, `Quaternion`, `Object3D`). Stock three converts
   quaternion→euler on *every* quaternion write to keep `object.rotation` in sync — for animated
   skeletons that's thousands of trig conversions per frame for values that are rarely read. The
   patch marks the euler dirty on quaternion writes and converts on first read. All Euler
   accessors and methods sync on demand, so behavior is unchanged. The conversion itself is also
   rewritten to compute the nine rotation-matrix elements inline instead of building a temporary
   `Matrix4` (bit-identical output).

2. **Render-list traversal fast paths** (`Renderer._projectObject`). Each object's render kind
   (mesh / light / group / …) is classified once and cached as a small integer instead of probing
   a chain of `isMesh`/`isLight`/… properties on every object every frame. Non-renderable
   containers skip the layers test, and leaf bones — the bulk of a skeleton-heavy scene graph —
   skip the recursive call entirely, both here and in `updateMatrixWorld`/`updateWorldMatrix`.

3. **Per-draw lookup memo** (`RenderObjects.get`). Resolving an object's `RenderObject` walked a
   4-level WeakMap chain (object → material → renderContext → lightsNode) on every draw. The last
   resolution is memoized on the 3D object and reused while the inputs are identity-equal
   (invalidated on dispose).

4. **Fused skeleton flatten** (`Skeleton.update`). The per-bone `multiplyMatrices` + `toArray`
   pair is fused into one loop writing directly into `boneMatrices`, removing an intermediate
   matrix store/load per bone per frame.

### Memory leak fix (WebGPU renderer)

Stock three registers a `dispose` listener **on the material** for every `RenderObject`, and only
material disposal removes it. With shared, long-lived materials (the common pattern), every
despawned mesh's render object stays pinned in the material's listener array forever — retaining
its geometry, skeleton and bindings. Under entity churn this grows unbounded (tens of MB per
minute in our workload). The patch:

- disposes the render object when its **geometry** is disposed (it is recreated automatically if
  the object is rendered again), and
- holds the render object via a **`WeakRef`** in the material's dispose listener, so shared
  materials can never pin dead render objects.

## How it was measured

Paired A/B harness: pristine npm `three@0.185.0` vs the patched build, alternated A/B/B/A with a
fresh page and dev-server per measurement, a fixed CPU-spin calibration before each run (rounds
where the machine throttled are excluded), comparing frame/render p50 — the patch won every
clean paired round across capped, uncapped and CPU-isolated (Node) configurations. The leak fix
was verified with heap snapshots and `FinalizationRegistry` (listener counts flat, despawned
meshes actually collected).

## Caveats

- Version-locked to `three@0.185.0`; other versions need the patch regenerated.
- Code that writes `object.rotation`'s internal `_x/_y/_z` fields directly (bypassing the public
  accessors) would read stale values — public API usage is fully covered.
- The geometry-dispose behavior change means disposing a geometry mid-use tears down its render
  object (it rebuilds on the next frame); stock kept it alive. Disposing in-use geometries is
  already undefined-ish behavior in three.
