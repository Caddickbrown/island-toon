# island-toon

Toon shading + cel outline post-processing for [Island Voxel Engine](https://github.com/Caddickbrown/Island-Voxel). Designed to be imported as ES modules — no bundler required.

Inspired by the visual style of [messenger.abeto.co](https://messenger.abeto.co/).

## What's in here

| File | What it does |
|------|-------------|
| `src/gradient.js` | Creates `DataTexture` gradient maps for `MeshToonMaterial` |
| `src/toon-material.js` | Cached toon material factory (replaces `MeshStandardMaterial`) |
| `src/normal-pass.js` | Renders the scene to a normal G-buffer for edge detection |
| `src/outline-pass.js` | Custom `ShaderMaterial` pass: depth + normal edge detection |
| `src/composer.js` | Wires up `EffectComposer`: RenderPass → OutlinePass → OutputPass |
| `demo.html` | Live demo with tunable sliders |

## How the outline works

Three.js's built-in `OutlinePass` only catches silhouette edges (outer object boundaries) via stencil dilation. It misses interior edges — the corners of window recesses, roofline details, tree trunk ridges.

This custom pass samples both **depth** and **normals** at 4 neighbouring pixels:

- **Depth discontinuity** → silhouette (object boundary)
- **Normal discontinuity** → surface detail (corner, window recess, etc.)

Both are composited into a single edge mask applied in one shader pass.

## Quick start — demo

```bash
cd island-toon
python3 -m http.server 8910
# open http://localhost:8910/demo.html
```

Use the sliders to tune outline thickness, depth/normal thresholds, and gradient step count live.

## Integration into Island Voxel Engine

### 1. Replace chunk material in `engine/renderer.js`

```js
import { toonMaterialFor } from '../../island-toon/src/toon-material.js';
import { GRAD3 } from '../../island-toon/src/gradient.js';

// Instead of:
// mesh.material = new THREE.MeshStandardMaterial({ color: ... });

// Use:
mesh.material = toonMaterialFor(VCOLOR[chunkPrimaryType], GRAD3);
```

`MeshToonMaterial` does **not** use `roughness` or `metalness` — remove those if present.

### 2. Replace the renderer call in `index.html` / game loop

```js
import { createComposer, renderFrame } from '../../island-toon/src/composer.js';

// Setup (once):
const { composer, normalTarget, outlinePass } = createComposer(renderer, scene, camera, {
  outlineColor:    0x1a1a1a,  // dark outline
  outlineThickness: 1.0,
  depthThreshold:  0.002,
  normalThreshold: 0.3,
});

// Each frame instead of renderer.render(scene, camera):
renderFrame(renderer, composer, scene, camera, normalTarget);
```

### 3. Fog to match sky colour

```js
const SKY = 0x7ed6cb;
scene.fog = new THREE.Fog(SKY, 60, 180);
scene.background = new THREE.Color(SKY);
```

## Tuning guide

| Parameter | Low | High | Effect |
|-----------|-----|------|--------|
| Gradient steps | 2 | 4 | 2 = harsh graphic, 4 = softer toon |
| Depth threshold | 0.0005 | 0.01 | Lower = more depth edges (can be noisy) |
| Normal threshold | 0.1 | 0.8 | Lower = more surface detail edges |
| Thickness | 0.5 | 3.0 | Outline width in pixels |
| Strength | 0 | 1 | Fade outlines in/out |

## Known pitfalls

- `gradientMap` **must** use `NearestFilter` — linear filtering blurs the steps
- `MeshToonMaterial` ignores `roughness`/`metalness` — it's not PBR
- The normal pass uses `scene.overrideMaterial` — any object with transparent or custom material will render normals incorrectly. Either skip those objects or handle them separately
- Water plane looks good without toon (use a `ShaderMaterial` for animated water, not `MeshToonMaterial`)
- Three.js r162+ required (`DepthTexture` on `WebGLRenderTarget`)

## Three.js version

Targets Three.js **r162+** (what Island Voxel uses). Demo loads r180 from jsDelivr.
