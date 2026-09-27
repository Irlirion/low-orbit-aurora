# DESIGN.md — Low Orbit Aurora (WebGL Hero)

Design reference for `index.html` — a single-file, full-viewport WebGL2 hero for a
Space / Low Orbit Internet website. This file documents the visual system, the
shader scene, the performance contract, and the tuning points. Edit the artifact
against these rules; when in doubt, the SpaceX-derived system below wins.

---

## 1. Concept

The interface disappears behind the imagery. A real-time aurora field plays
full-screen over pure black — spectral blue-violet curtains, a parallax
starfield, and the curved, unlit limb of Earth — while a stenciled uppercase
type block and one ghost button sit directly on the scene. No cards, no panels,
no shadows, no second color.

Every section of the page is one cinematic frame: photography in the original
system, GPU-rendered light here.

## 2. Visual direction

Derived from the SpaceX design system (achromatic, photographic, industrial):

| Rule | Application in this artifact |
|---|---|
| Pure black canvas | `--bg: #000000`; the shader never paints above the tonemap's intent — the void stays void |
| Spectral white, never pure white | All text is `#f0f0fa`; hover alone reaches `#ffffff` |
| Achromatic + spectral tint only | The aurora's one hue is the blue-violet tint already present in `#f0f0fa`; no neon green, no chromatic accent |
| Universal uppercase + positive tracking | Every text element is uppercase with ≥ 0.96px tracking |
| One ghost button | The 32px pill CTA is the only rounded element and the only primary action |
| Zero shadows / zero cards | Depth comes from the scene; overlays are gradients, not surfaces |
| One decisive flourish | The aurora itself — nothing else competes |

## 3. Tokens (bound in `:root`, copy verbatim when forking)

### Color
| Token | Value | Role |
|---|---|---|
| `--bg` / `--surface` | `#000000` | Page canvas (no card tier exists) |
| `--fg` | `#f0f0fa` | Spectral White — headings, CTA, kicker |
| `--muted` | `rgba(240,240,250,0.7)` | Sub-line, HUD status |
| `--border` | `rgba(240,240,250,0.35)` | Ghost CTA edge, HUD toggle edge |
| `--border-soft` | `rgba(240,240,250,0.1)` | HUD status hairline |
| `--accent-hover` | `#ffffff` | Hover foreground |
| `--accent-active` | `#d8d8e6` | Pressed foreground |
| `--focus-ring` | `0 0 0 2px var(--accent)` | `:focus-visible` on all controls |

No raw hex outside the `:root` block. No second hue anywhere in CSS.

### Typography
| Role | Font | Size | Weight | Tracking | Line-height |
|---|---|---|---|---|---|
| Hero | `--font-display` (D-DIN-Bold → DIN Alternate → Helvetica Neue → Arial) | `clamp(32px, 4.6vw, 48px)` | 700 | `0.02em` | 1.0 |
| Kicker | `--font-body` (D-DIN) | 13px | 700 | 1.17px | 0.94 |
| Sub | `--font-body` | 16px (14px ≤ 600px) | 400 | 0.3px | 1.6 |
| CTA / toggle | `--font-body` | 13px / 10px | 700 | 1.17px / 1px | 0.94 |
| HUD micro | `--font-body` | 10px | 400 | 1px | 0.94 |

Hero `max-width: 14ch` (12ch on phones) so "INTERNET FROM ORBIT" breaks into
two balanced lines with no orphan characters.

### Spacing, radius, motion
- Scale: 4 / 8 / 12 / 16 / 20 / 24 / 32 / 48px (`--space-*`).
- Radius: `4px` for sharp utility (HUD chips), `32px` for the ghost CTA — only rounded element.
- Motion: `150ms cubic-bezier(0.2, 0, 0, 1)` on all state transitions; nothing else animates in CSS.

## 4. Layout & composition

```
canvas #gl          z1  fixed inset 0 — the scene (role="img" + aria-label)
.overlay            z2  fixed inset 0 — legibility gradients + hero stack (pointer-events: none)
.hud                z3  fixed top-right — telemetry chip + motion toggle
```

- **Hero stack** (bottom-left, flex-end): kicker → h1 → sub → ghost CTA →
  fallback notice (hidden unless WebGL fails). Only `.btn-ghost` re-enables
  `pointer-events`.
- **Legibility gradients** on `.overlay` (text sits above them):
  vertical `rgba(0,0,0,0.75) 0% → 0.5 32% → 0 60%`,
  horizontal `rgba(0,0,0,0.5) → 0 55%`. Text zones resolve to near-black
  backgrounds; the sub-line holds ≈ 7:1 contrast.
- **HUD**: status chip (`rgba(0,0,0,0.6)` + hairline border) and the Pause/Resume
  toggle (ghost surface, sharp 4px radius, 44px min target). Sharp radius keeps
  it visually subordinate to the pill CTA. No `backdrop-filter` — it forced a
  per-frame re-blur of the animating canvas.
- **Responsive**: ≤ 960px hero caps at 40px; ≤ 600px padding 18px, hero ≤ 32px,
  status chip hidden (toggle remains), no horizontal scroll ever.

## 5. The WebGL scene

Fullscreen triangle, fragment shader (`#version 300 es`), `alpha:false`,
`depth:false`, `stencil:false` (no unused buffers), DPR clamped, redraw every
rAF. Layers painted back to front:

| # | Layer | Construction | Key constants |
|---|---|---|---|
| 1 | Starfield | 2 parallax hash-grid layers, per-star twinkle, dimmed under curtains | scales `46 / 92`, thresholds `0.950 / 0.976` |
| 2 | Aurora curtains | domain-warped `fbm3` + `fbm4` curtains + ray noise; striation *derived* from the same ray field | band `smoothstep(0.45, 0.80)` with `rays×0.10`, vertical mask `-0.34…-0.10` fading out `0.14…0.52`, striation `0.45 + 0.55×rays` |
| 3 | Secondary band | derived from `warp + rays` — no extra tap, narrow low-altitude mask | `smoothstep(0.58, 0.95, warp×0.85 + rays×0.30)`, weight `×0.4` |
| 4 | Earth limb | circle SDF at `(0,-2.2) r=1.85`; interior forced to black, atmosphere rim above | rim `exp(-ed·11.0)×0.70` tint + `exp(-ed·34.0)×0.50` spectral |
| 5 | Finish | vignette → Reinhard-ish tonemap → gamma lift | vignette `0.38·dot`, `col/(1+0.55·col)`, `pow(col, 0.90)` |

Palette inside the shader (linear-ish RGB literals — the only colors outside `:root`):

```glsl
vec3 spectral = vec3(0.9412, 0.9412, 0.9804); /* #f0f0fa */
vec3 tint     = vec3(0.50, 0.56, 1.0);         /* blue-violet spectral tint */
```

Aurora cores mix `tint → spectral` at `aur × 0.70`, so the brightest rays go
near-white while the body keeps the violet cast. **Do not introduce green** —
the palette stays achromatic with spectral tint.

### Pointer reaction (hover interactivity)

Two uniforms drive it: `mouse` (smoothed pointer in shader uv-space) and
`mforce` (0…1 influence) = `presence × (0.30 + 0.70 × energy)`.

| Effect | Construction | Constant |
|---|---|---|
| Depth parallax | `par = mouse × mforce × 0.02` — stars take `×0.5`, aurora `×1.0`, Earth `×0.85` | max ≈ 18px lean @1080p |
| Cursor halo | `exp(-dist × 11.0) × mforce`, composited *before* the Earth mask so the planet occludes it | weight `×0.10` |
| Ray boost | cursor field feeds the curtain band: `smoothstep(0.45, 0.80, curtains + rays×0.10 + mg×0.35)` | local band coverage swells while sweeping |
| Core flare | aurora cores gain `aur × mg × 0.35` | transient, only near the pointer |

JS side: position smoothed at τ ≈ 100ms, presence τ ≈ 170ms; sweep energy
saturates after roughly a third of a screen-height of travel and decays at
τ ≈ 450ms; `pointerdown` injects an instant pulse. Leaving the window fades
influence to zero (the glow dissipates in place, it does not jump to center).
Cost: one `exp` + scalar math — **noise budget stays 8 taps**.
Under `prefers-reduced-motion` the field snaps to the pointer with a constant
`0.30 × presence` glow (no energy ramp); while paused, pointer reaction is
frozen with the rest of the scene. Initial state (no pointer yet) renders
identically to the non-interactive shader.

### Shader performance rules (hard constraints)
- `hash` is sin-free (Hoskins-style). Never reintroduce `fract(sin(dot(…)))` —
  it ran ~88× per pixel.
- Total noise budget: **8 taps/pixel** (`warp 3 + curtains 4 + rays 1`).
  The striation and secondary band are *derived* from `rays`/`warp` — adding a
  layer means removing a tap. The haze tap was removed (it contributed <1.5%
  luminance); `warp` is 3 octaves, `curtains` stays 4 (the hero silhouette).
- All `smoothstep` edges must be strictly increasing — reversed edges are
  undefined behavior in GLSL (two were caught and fixed during review).

## 6. Performance contract (target: stable 60+)

The badge (`GPU · WebGL2 · Nfps`) reports a 1-second rolling measurement.

| Mechanism | Rule |
|---|---|
| Context flags | `antialias:false, alpha:false, depth:false, stencil:false` — no unused render buffers |
| Render scale start | `quality = 0.75` on HiDPI (≈1.5 effective DPR), `1.0` on 1× displays; hard cap DPR 2 |
| Step down | 2 consecutive windows `< 59fps` → `quality × 0.8`, floor `0.5` |
| Hard rescue | a single window `< 45fps` steps down immediately, 1500ms cooldown (weak GPUs reach the floor fast) |
| Probe up | 8 consecutive windows `≥ 59.5fps` → `quality × 1.12`, cap `1.0` |
| Cooldown / warm-up | 2500ms between normal changes; no changes in the first 3000ms (startup jank immunity) |
| Reset points | Pause/Resume click resets FPS counters and streaks |
| `gl.getError()` | first 5 draws only (each call can stall the pipeline) |
| Compositor | no `backdrop-filter`, no CSS blur over the canvas; zero box-shadows |
| Reduced motion | `prefers-reduced-motion` starts paused; a single frame is drawn, loop idles until resize |

If fps settles below target on a weak GPU, the controller trades resolution,
never frame pacing. The floor (0.5) equals 1.0 effective DPR on retina and 0.5
on 1× displays — below that, sharpness loss outweighs the gain.

## 7. Interaction states & accessibility

| Element | Default | Hover | Active | Focus |
|---|---|---|---|---|
| `.btn-ghost` (CTA) | bg `0.1`, border `0.35`, fg `--fg` | bg `0.2`, fg `#fff` | bg `0.06`, fg `--accent-active` | 2px spectral ring, outline none |
| `.hud-toggle` | same ghost surface, radius 4px | bg `0.2`, fg `#fff` | bg `0.06`, fg `--accent-active` | 2px spectral ring |

- Foreground only ever moves *toward* white on dark — contrast never drops in a state.
- **Scene reaction:** the canvas responds to the pointer — halo, ray boost, and
  parallax per §5.1. The CTA additionally lifts `translateY(-1px)` on hover
  (150ms, `transform` in its transition list) and settles on `:active`.
- Touch targets ≥ 44px (`min-height` on CTA, toggle, status chip).
- CTA is the single primary action; the HUD toggle is a sharp micro utility and
  doubles as the WCAG 2.2.2 pause control for ambient motion.
- `aria-pressed` on the toggle; canvas carries `role="img"` + aria-label.
- **Failure path:** no WebGL2 / shader error → `body.gl-failed` shows the static
  spectral-haze CSS fallback and a "Static mode" notice; never a blank page.

## 8. Content rules

- All copy renders uppercase via CSS; write plain sentence case in the HTML.
- Kicker names the domain, headline ≤ 14ch of visual width, one supporting line.
- One ghost CTA per viewport (`href="#network"` — placeholder anchor, replace
  with the real section when integrating).
- No invented metrics, no emoji, no icon decoration.

## 9. File map & maintenance

| File | Role |
|---|---|
| `index.html` | The artifact — self-contained (tokens, CSS, shader, JS in one file). Renamed from `low-orbit-aurora.html`; preserve the name on edit |
| `DESIGN.md` | This document |
| `low-orbit-aurora.html.artifact.json` | Stale sidecar from the original filename |

Inspectable hooks (`data-od-id`): `hud-telemetry`, `hud-status`,
`hud-motion-toggle`, `hero-overlay`, `hero-kicker`, `hero-title`, `hero-sub`,
`hero-cta`, `hero-fallback`.

**Tuning cheat-sheet** — most-likely edits and where:

| Wanted change | Location |
|---|---|
| Aurora coverage / brightness | `band = smoothstep(0.45, 0.80, …)` and `col += … × aur` (keep intensity ≤ ~1.0; the render check flagged 1.15 as washed) |
| Aurora hue | `tint` vector only — stay blue-violet |
| Horizon height / curvature | Earth SDF: `length(uv - vec2(0,-2.2)) - 1.85` |
| Rim glow reach | `exp(-ed × 11.0)` — higher decay = tighter rim; ≥ ~6 washed the text zone |
| Star density | `thr` values `0.950 / 0.976` (lower = more stars) |
| FPS behavior | `adaptQuality()` thresholds and `quality` bounds |
| Text safety | `.overlay` gradient stops — never lower the bottom black below 0.7 without rechecking sub-line contrast |
