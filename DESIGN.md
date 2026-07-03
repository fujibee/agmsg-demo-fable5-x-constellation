# AI Constellation Map — Detailed Design

**Project:** "What X is saying about AI models" rendered as an interactive 3D constellation map.
**Deliverable:** a single self-contained HTML file (`index.html`) using three.js. Drag to rotate, scroll to zoom, click a star to read the posts behind it.
**Author:** fable (design only — implementation by opus, data by grok, polish by codex)

---

## 1. Concept

The night sky is the metaphor. Each **AI model is a constellation**; each **topic being discussed on X is a star**. Star size encodes mention volume, star pulse/brightness encodes how hot the topic is right now, and thin constellation lines join the stars of one model into a recognizable figure. The whole sky slowly auto-rotates until the user grabs it.

Five constellations:

| id | name | vendor | color (hex) | color rationale |
|----|------|--------|-------------|-----------------|
| `fable-5` | Fable 5 | Anthropic | `#E8865A` | warm coral — Anthropic clay, protagonist warmth |
| `opus-4-8` | Opus 4.8 | Anthropic | `#F2C14E` | amber gold — sibling of coral, clearly distinct |
| `gpt-5-6` | GPT-5.6 | OpenAI | `#5CD6B9` | teal green — OpenAI heritage |
| `gemini-3` | Gemini 3 | Google | `#8B7CF6` | violet — Gemini gradient feel |
| `glm-5-2` | GLM 5.2 | Zhipu | `#4E9CFF` | clear blue — distinct from violet at a glance |

These five hues are deliberately spread around the wheel (orange / gold / teal / violet / blue) so clusters are identifiable in peripheral vision without reading labels.

---

## 2. Screen layout

Full-viewport WebGL canvas; all UI is HTML overlaid with absolute positioning. Dark UI over dark sky.

```
┌──────────────────────────────────────────────────────────────┐
│ ◤ AI CONSTELLATION                            ● Fable 5      │
│   What X is talking about · 2026-07-03        ● Opus 4.8     │
│   (top-left, title block)                     ● GPT-5.6      │
│                                               ● Gemini 3     │
│                                               ● GLM 5.2      │
│                                               (top-right,    │
│                  ✦     ·    ✦                  legend chips) │
│              ✦ ── ✦ ── ✦        ┌─────────────────────────┐  │
│                 CANVAS          │  DETAIL PANEL (hidden    │  │
│           ✦ ·    ✦    · ✦       │  until a star is clicked)│  │
│                                 │  360px, slides in right  │  │
│                                 └─────────────────────────┘  │
│ drag to rotate · scroll to zoom · click a star    (bottom-left)│
└──────────────────────────────────────────────────────────────┘
```

### 2.1 Title block (top-left, 28px inset)
- Line 1: `AI CONSTELLATION` — 13px, letter-spacing 0.3em, uppercase, `rgba(255,255,255,0.55)`.
- Line 2: `What X is talking about right now` — 22px, weight 600, `#F5F3EE`.
- Line 3: date from `data.generatedAt` — 12px, `rgba(255,255,255,0.35)`.
- Font stack: `Inter, "Helvetica Neue", system-ui, sans-serif` (no webfont download; system fallback is fine).

### 2.2 Legend (top-right, 28px inset)
One chip per model, stacked vertically, 8px gap:
- Chip: flex row, 10px color dot (model color, `box-shadow: 0 0 8px <color>`) + model name 13px `rgba(255,255,255,0.75)`.
- Hover: background `rgba(255,255,255,0.06)`, radius 8px.
- Click: fly camera to that cluster (same behavior as clicking its label in 3D).
- Double-click: toggle dim — other clusters drop to 15% opacity (stars, lines, labels). Double-click again to restore.

### 2.3 Detail panel (right side)
- 360px wide, `top:88px; bottom:24px; right:24px`, border-radius 16px.
- Glass: `background: rgba(12,16,28,0.72); backdrop-filter: blur(16px); border: 1px solid rgba(255,255,255,0.08)`.
- Hidden state: `transform: translateX(400px)`; shown: `translateX(0)`; transition `transform 380ms cubic-bezier(0.22,1,0.36,1)`.
- Content top→bottom:
  1. Model chip (dot + vendor · model name, 12px, model color).
  2. Topic label — 20px weight 700 `#F5F3EE`.
  3. Meta row: `▲ 1,250 mentions` + heat meter (60px bar, fill = `heat`, gradient from model color to white, `border-radius 999px`).
  4. Summary — 14px/1.6 `rgba(255,255,255,0.72)`.
  5. `POSTS` divider — 11px uppercase letter-spacing 0.2em `rgba(255,255,255,0.35)`.
  6. Post cards (one per post): `background rgba(255,255,255,0.04); border-radius 12px; padding 12px`. Author `@handle` 13px weight 600 in model color; text 13px/1.55 `rgba(255,255,255,0.8)`; footer `♥ 3.4k · ⇄ 900` 11px `rgba(255,255,255,0.4)`. Whole card is `<a target="_blank">` to `post.url`.
  7. Close button `×` top-right of panel, 28px hit area.

### 2.4 Hint bar (bottom-left, 24px inset)
`drag to rotate · scroll to zoom · click a star` — 12px `rgba(255,255,255,0.35)`. Fade out (opacity 0, 1s transition) after first pointerdown on canvas.

### 2.5 Hover tooltip
Small floating label following the cursor (offset +14px,+14px): topic label, 12px, glass pill (`rgba(12,16,28,0.85)`, radius 999px, padding 6px 12px, 1px border model-color at 40%). Shows on star hover only.

---

## 3. The sky (background)

Layered, cheap, atmospheric:

1. **Clear color** `#05070D` (not pure black — deep navy reads richer).
2. **Nebula gradient dome**: `THREE.SphereGeometry(1200, 32, 32)` with `BackSide` ShaderMaterial — vertical gradient from `#0A0E1E` (bottom) through `#05070D` (mid) to `#0B0A18` (top), plus two very faint radial color blooms baked into the fragment shader (violet `#1A1440` at 6% around one pole direction, teal `#0E2A33` at 4% opposite). No texture download needed.
3. **Background starfield**: 2600 points in a `THREE.Points` cloud, positions uniform on a shell radius 700–1000. Sizes 0.6–1.8, color white with slight blue/warm tint variation (±8% hue), opacity 0.35–0.9. `sizeAttenuation: true`, additive blending, depthWrite false. These never raycast (set `raycast = noop` or keep in a separate non-interactive group).
4. **Fog**: `scene.fog = new THREE.Fog(0x05070D, 500, 1100)` — softens the dome and far stars.

---

## 4. Stars (topics)

### 4.1 Geometry & material
Each topic star = `THREE.Sprite` with a shared canvas-generated texture, per-sprite color/scale. Sprites beat meshes here: perfect billboards, cheap glow, one draw call per star is acceptable at ≤40 stars.

**Star texture** — `createStarTexture()`: 256×256 canvas, radial gradient:
- 0.00–0.08: `rgba(255,255,255,1)` (white-hot core)
- 0.08–0.30: `rgba(255,255,255,0.9)` → color-tintable falloff (draw pure white; tint via `sprite.material.color`)
- 0.30–1.00: exponential falloff to transparent (`alpha = pow(1-t, 2.5)`)
- Plus 4 diffraction spikes: two crossed lines (horizontal/vertical), 2px wide, gradient alpha 0.35 → 0, length 0.9× radius. This gives the "telescope photo" sparkle.

`SpriteMaterial({ map: starTex, color: modelColor, blending: THREE.AdditiveBlending, depthWrite: false, transparent: true })`. One material **per model** (5 materials), shared by its stars; per-star variation via scale and a per-star `userData.phase`.

**Halo layer**: behind each star sprite, a second sprite with a softer texture (pure radial `pow(1-t,3.5)`, no spikes) at 2.6× the star's scale and opacity `0.22 + 0.5*heat`, same color. This is the glow — visible even without post-processing. (codex may later replace/augment with UnrealBloomPass; the halo must look good alone.)

### 4.2 Size mapping (volume → scale)
```
minV = min(volume of all topics), maxV = max(...)
t = (log(volume) - log(minV)) / (log(maxV) - log(minV))   // 0..1, log scale
scale = 7 + t * 17        // world units: 7 (small) … 24 (dominant topic)
```

### 4.3 Heat mapping (heat → life)
`heat` ∈ [0,1] drives the pulse in the render loop:
```
pulse = 1 + heat * 0.12 * sin(elapsed * (0.8 + heat*1.6) + phase)
sprite.scale.setScalar(baseScale * hoverBoost * pulse)
halo.material  // per-star halo opacity flickers: haloBase * (0.9 + 0.1*sin(elapsed*2.3 + phase))
```
`phase = hash(topic.id) * 2π` so stars never pulse in sync. Hot topics (heat ≥ 0.8) visibly breathe; cold ones are almost static. Subtlety is the point — max ±12% scale.

### 4.4 Hover / selected states
- Hover: `hoverBoost` lerps 1 → 1.45 at 12%/frame (`boost += (target-boost)*0.12`); halo opacity ×1.5; tooltip shows; cursor `pointer`.
- Selected: `hoverBoost` target 1.6, plus a thin ring sprite (circle stroke texture, model color, opacity 0.8, scale 1.9× star) that slowly rotates (0.3 rad/s) — the "you are here" marker. Only one star selected at a time.

---

## 5. Constellations (clusters)

### 5.1 Cluster placement — pentagon-on-a-sphere
5 cluster centers on a sphere of radius **R = 150** around the origin, alternating above/below the equator so no two neighbors share a plane (avoids a flat ring look):

```
for i in 0..4:
  lon = i * 72° + 20°            // constant offset so no cluster sits exactly on an axis
  lat = (i % 2 == 0) ? +18° : -22°
  center[i] = spherical(R, lat, lon)
```
Model order around the ring: `fable-5, gpt-5-6, opus-4-8, gemini-3, glm-5-2` — the two Anthropic (warm) constellations are never adjacent, keeping color contrast between neighbors.

### 5.2 Star placement within a cluster
Deterministic, seeded by topic id — same data always renders the same sky. Use mulberry32 seeded with a string hash (`fnv1a(topic.id)`).

```
layoutTopics(cluster):
  sort topics by volume desc
  for rank, topic in topics:
    // importance pulls big stars toward the cluster heart
    w   = 1 - rank / max(1, n-1)              // 1 = biggest … 0 = smallest
    r   = 8 + pow(1 - w, 0.75) * 34           // radius 8..42 from center
    dir = random unit vector (seeded), then flattened:
          dir.y *= 0.55                        // ellipsoid — constellations read better squashed
    pos = center + normalize(dir) * r
    // min-separation: retry up to 40 seeded jitters until
    // dist(pos, any placed star) >= (scaleA + scaleB) * 0.9 + 6
```
Cluster bounding radius ≈ 45; with R = 150 and 72° spacing, inter-cluster gaps stay ≥ 60 units — clusters never visually merge.

### 5.3 Constellation lines — minimum spanning tree
Real constellations are sparse line figures, not hairballs. Build a **Prim's MST** over each cluster's stars using 3D euclidean distance, then draw each MST edge:

- `THREE.LineSegments`, `LineBasicMaterial({ color: modelColor, transparent: true, opacity: 0.28, blending: THREE.AdditiveBlending, depthWrite: false })`.
- One `LineSegments` object per cluster (so dim-toggling is per-cluster trivial).
- Edge endpoints pulled back 55% of each star's scale so lines don't pierce star cores: `a' = a + normalize(b-a) * (scaleA*0.55)`, symmetric for `b'`.
- On cluster focus (camera flown to it): tween that cluster's line opacity 0.28 → 0.5; others → 0.12.

### 5.4 Cluster labels
Canvas-texture sprite per cluster at `center + (0, clusterRadius + 14, 0)`:
- Texture: 512×128 canvas — model name 44px weight 700 white, below it vendor 22px at 55% alpha, subtle color underline bar (model color, 4px, width of text).
- `SpriteMaterial({ depthWrite:false, transparent:true })`, scale ≈ (48, 12).
- Opacity by camera distance: full at ≤ 350, fades to 0.35 at 600 (labels shouldn't shout when zoomed out to the whole sky... actually invert: full when far, 0.6 when near a *different* cluster; keep it simple: `opacity = clamp(mapLinear(distToCamera, 150, 600, 0.55, 1.0))`).
- Raycastable via an invisible `PlaneGeometry` proxy or by including sprites in the raycast list; click = focus cluster.

---

## 6. Camera & interaction

### 6.1 Camera
- `PerspectiveCamera(55, aspect, 0.1, 3000)`, initial position `(0, 60, 330)`, lookAt origin.
- `OrbitControls`:
  - `enableDamping: true, dampingFactor: 0.06` (heavy, planetarium feel)
  - `rotateSpeed: 0.55`, `zoomSpeed: 0.9`
  - `minDistance: 120, maxDistance: 620`
  - `enablePan: false` (the sky has one center; panning gets users lost)
  - `autoRotate: true, autoRotateSpeed: 0.35`

### 6.2 Idle auto-rotate
Any `pointerdown`/`wheel` sets `autoRotate = false` and stamps `lastInteraction`. In the loop, if `now - lastInteraction > 7s` and no panel is open, ease it back on (lerp `autoRotateSpeed` 0 → 0.35 over ~2s to avoid a jerk).

### 6.3 Raycasting
- `Raycaster` with `params.Sprite` default; maintain `interactive = [...starSprites, ...labelSprites]`.
- On `pointermove` (throttled to rAF): raycast; nearest star sprite → hover state; else clear.
- On `click` (only if pointer moved < 5px between down/up — don't fire after a drag):
  - hit star → `selectStar(topic)`
  - hit label → `focusCluster(model)`
  - hit nothing → `deselect()`
- `Escape` key → `deselect()`. Double-click empty space → `resetCamera()`.

### 6.4 Camera tweens (no external lib — 20-line helper)
```
tweenCamera(toPos, toTarget, ms=900):
  cubic ease-in-out on t∈[0,1]
  camera.position = lerpVectors(fromPos, toPos, e(t))
  controls.target  = lerpVectors(fromTarget, toTarget, e(t))
  controls disabled during tween, re-enabled after
```
- `selectStar`: target = star position; camera position = star pos + direction(camera→star reversed) scaled so distance = 90, biased +12 in y. Panel opens at tween start (they animate together). Auto-rotate off while panel open.
- `focusCluster`: target = cluster center; distance 170.
- `resetCamera`: back to `(0, 60, 330)` / origin, 1100ms.

### 6.5 Resize & DPR
`renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`; `resize` handler updates camera aspect + renderer size. No layout thrash — canvas is `position:fixed; inset:0`.

---

## 7. Data — JSON schema (grok fills this)

Embedded in the HTML as `const DATA = {...}` (single-file constraint; no fetch). grok delivers this exact shape to opus:

```json
{
  "generatedAt": "2026-07-03T18:00:00Z",
  "source": "x.com",
  "models": [
    {
      "id": "fable-5",
      "name": "Fable 5",
      "vendor": "Anthropic",
      "color": "#E8865A",
      "topics": [
        {
          "id": "fable-5-agentic-coding",
          "label": "Agentic coding leap",
          "summary": "One or two sentences summarizing what X is saying about this topic.",
          "volume": 1250,
          "heat": 0.85,
          "posts": [
            {
              "author": "@handle",
              "text": "Post text, may be truncated to ~280 chars.",
              "url": "https://x.com/handle/status/123456789",
              "likes": 3400,
              "reposts": 900,
              "postedAt": "2026-07-03T12:34:00Z"
            }
          ]
        }
      ]
    }
  ]
}
```

**Contract (grok MUST follow):**
- Exactly 5 models, with these exact `id`/`name`/`vendor`/`color` values (see table in §1).
- 4–8 topics per model. Topic `id` = model id + kebab-case slug, unique across the file.
- `label` ≤ 40 chars (it becomes a tooltip and panel title). `summary` 1–2 sentences.
- `volume`: integer ≥ 1 — relative mention count. Consistent methodology across all topics (sizes are compared globally).
- `heat`: float 0–1 — how actively it's being discussed *right now* (velocity + recency + engagement). At least one topic overall should be ≥ 0.9 and a few ≤ 0.3, so the pulse contrast shows.
- `posts`: 2–4 representative real posts, sorted by engagement desc. `url` must be a real `https://x.com/...` status link. `likes`/`reposts` integers (0 allowed). Escape properly — this lands inside a `<script>` block (no raw `</script>` in text).

---

## 8. three.js implementation structure (opus)

Single `index.html`. three.js via ESM CDN import map (pin a version):

```html
<script type="importmap">
{ "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.166.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.166.0/examples/jsm/"
} }
</script>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
```

File layout inside the single HTML: `<style>` (all CSS), `<div id="ui">` (title, legend, hint, tooltip, panel), `<canvas>` — then one `<script type="module">` organized as:

```
// ---------- data ----------
const DATA = {...};                        // grok's JSON verbatim
const MODEL_ORDER = ['fable-5','gpt-5-6','opus-4-8','gemini-3','glm-5-2'];

// ---------- utils ----------
fnv1a(str) -> uint32           // string hash for seeds
mulberry32(seed) -> () => f32  // seeded RNG
easeInOutCubic(t)
lerpV3(a, b, t, out)

// ---------- textures ----------
createStarTexture()            // §4.1 core + spikes
createHaloTexture()            // §4.1 soft radial
createLabelTexture(name, vendor, color)  // §5.4
createRingTexture()            // §4.4 selection ring

// ---------- layout ----------
layoutClusters(models) -> Map<modelId, {center: V3}>          // §5.1
layoutTopics(model, center) -> [{topic, pos: V3, scale}]      // §4.2 + §5.2
computeMST(points) -> [[i, j], ...]                            // §5.3 Prim's

// ---------- scene build ----------
initRenderer()                 // renderer, camera, controls, resize, fog
buildSky()                     // §3 dome + background starfield
buildConstellations(DATA)      // per model: group { stars[], halos[], lines, label }
                               //   sprite.userData = { topic, model, baseScale, phase, halo }

// ---------- interaction ----------
onPointerMove(e) / onClick(e)  // §6.3 raycast; drag-vs-click via pointer delta
selectStar(sprite) / deselect()
focusCluster(modelId)
tweenCamera(pos, target, ms)   // §6.4
renderPanel(topic, model)      // fills #panel DOM from data
renderLegend()                 // builds legend chips, wires click/dblclick

// ---------- loop ----------
function animate(tMs):
  requestAnimationFrame(animate)
  const t = tMs / 1000
  controls.update()
  updatePulses(t)              // §4.3 scale + halo flicker on every star
  updateHover()                // lerp hoverBoost toward targets
  updateLabelOpacity()         // §5.4 distance fade
  updateIdleAutorotate()       // §6.2
  spinSelectionRing(t)
  renderer.render(scene, camera)
```

**Definition of done (opus):**
1. Opens from `file://` or any static server with no console errors, 60fps on a laptop.
2. Whole sky auto-rotates; drag/zoom feel damped; idle resumes rotation.
3. 5 labeled constellations, MST lines, log-scaled pulsing stars per the data.
4. Hover tooltip + cursor; click star → camera fly + panel with real posts (links open X in new tab); ESC/off-click deselects; legend chips focus clusters.
5. Data is grok's JSON pasted as `DATA` — zero code changes needed when data updates.

---

## 9. Polish targets (codex, after opus)

Priority order — stop where diminishing returns start, keep 60fps:

1. **Bloom**: `EffectComposer` + `UnrealBloomPass` (strength 0.55, radius 0.6, threshold 0.72). Re-tune halo opacities down (~×0.6) so cores don't blow out. If FPS drops on hidpi, render composer at DPR 1.5 max.
2. **Inertia audit**: star hover lerp, panel easing, camera tween — nothing may snap. Add a 150ms opacity fade for the tooltip.
3. **Line quality**: replace `LineBasicMaterial` (1px limit) with fatter feel — either `Line2`/`LineMaterial` (addons) at ~1.6px, or per-edge gradient opacity (brighter near big stars).
4. **Micro-parallax**: background starfield rotates at 0.35× the camera's effective angular velocity (or just a slow constant counter-drift, 0.002 rad/s) — depth without cost.
5. **Star twinkle**: ±6% random opacity shimmer on background points via a per-point phase attribute in a tiny ShaderMaterial (only if cheap).
6. **Panel niceties**: heat meter animates fill on open; post cards stagger in (40ms/card, translateY 8px → 0); custom scrollbar (6px, `rgba(255,255,255,0.15)`).
7. **Entrance choreography**: on load, stars fade/scale in cluster-by-cluster over ~1.6s total (100ms stagger between clusters), lines draw in after (opacity 0 → target). Sets the mood immediately.
8. **Mobile sanity**: touch rotate/zoom work (OrbitControls default); panel becomes bottom sheet under 640px width (`bottom:0; left:0; right:0; height:55vh; translateY` instead of X).

---

## 10. Team flow

```
fable (this doc) ──schema──▶ grok ──DATA json──▶ opus ──index.html──▶ codex ──▶ fable (visual review) ──▶ koichi
```

- **grok**: gather X topics per §7 contract → send the JSON to opus via agmsg (or write `data.json` in this directory and notify).
- **opus**: implement §1–§8 as `index.html` in this directory. If grok's data hasn't arrived, build with placeholder data of the exact schema, swap on arrival. When done → notify codex.
- **codex**: apply §9 to `index.html` (edit in place) → notify fable.
- **fable**: visual review, change requests if needed, then deliver to koichi.
