---
name: r3f-galaxy-scroll-background
description: |
  Generates a cinematic, scroll-driven 3D galaxy background using React Three Fiber
  and Three.js — spiral galaxy particle field, orbiting planets, and drifting meteoroids,
  with the camera zooming into the galaxy core as the user scrolls, dissolving into the
  page content underneath. Use this skill whenever the user asks for a "space background",
  "galaxy intro", "hero background with planets", "cosmic scroll effect", "zoom into space
  on scroll", a portfolio/landing-page welcome screen with stars/planets/meteors, or any
  request that mentions combining Three.js particles with GSAP ScrollTrigger camera movement.
  Also trigger this skill for requests like "make my intro feel like flying through space"
  or "3D solar system hero section", even if the user doesn't use the word "galaxy" explicitly.
---

# React Three Fiber — Galaxy Scroll Background

Builds a layered, performant 3D space scene (spiral galaxy + planets + meteoroids) that
reacts to scroll by zooming the camera toward the galaxy core, then handing off smoothly
to the rest of the page. Designed for hero sections, portfolio intros, and landing pages
that want a premium, cinematic first impression without relying on heavy external assets.

## When to use this skill

Use this skill for any request involving:
- A full-screen or hero-section space/galaxy/cosmic background
- Planets, meteors, asteroids, or a starfield that needs to feel "alive" (drifting, rotating)
- A scroll-triggered camera zoom (flying into a scene, entering a portal, "diving" into content)
- A cinematic intro screen that transitions into a normal webpage below it

Do NOT use this skill for: static wallpaper images, 2D CSS-only starfields (use plain CSS/SVG
for those, it's cheaper), or scenes requiring scientifically accurate astronomy (this skill
optimizes for visual impact, not accuracy).

## Core architecture (4 layers)

Build the scene as four independent, composable layers inside a single `<Canvas>`. Keep
each layer as its own component so they can be toggled, reused, or dropped into other
projects independently.

### Layer 1 — Spiral galaxy particle field

The foundation. Generate particles procedurally (never load a texture/model for this) using
the standard "Galaxy Generator" approach popularized by Three.js Journey:

- Use `THREE.BufferGeometry` with a `Float32Array` position attribute, NOT individual meshes
  per particle — thousands of individual meshes will tank performance. One `Points` object
  with one `BufferGeometry` handles tens of thousands of particles at 60fps.
- Distribute particles along spiral arms using polar coordinates: pick a random `radius`,
  compute a `spinAngle = radius * spinFactor`, add a random `branchAngle` for however many
  arms you want (typically 3-5), then add randomized offset (`randomX/Y/Z`) scaled down as
  radius increases so the core is denser and arms are looser.
- Color particles with a gradient from a bright "core" color to a dimmer "outer arm" color
  using `THREE.Color.lerpColors(insideColor, outsideColor, radius / maxRadius)` per-particle,
  stored in a `color` BufferAttribute.
- Material: `THREE.PointsMaterial` with `sizeAttenuation: true`, `vertexColors: true`,
  `blending: THREE.AdditiveBlending`, `depthWrite: false`, `transparent: true`. Additive
  blending is what makes overlapping particles glow instead of looking like flat dots.
- **Circular particles, not squares**: `PointsMaterial` renders square points by default.
  Generate a small radial-gradient circle on an offscreen `<canvas>` at runtime, convert it
  to a `THREE.CanvasTexture`, and assign it to both `map` and `alphaMap` on the material.
  Never ship a static PNG for this — the runtime-generated texture is a few lines of code
  and keeps the skill dependency-free.
- Animate slow rotation of the whole galaxy group in `useFrame` (rotate the parent `<group>`
  or `<points>` on its Y axis at a tiny increment like `0.0003-0.0006` per frame) — this alone
  sells the "living galaxy" effect more than any other single choice.

### Layer 2 — Planets

2-4 planets is the sweet spot; more than that clutters a hero section and hurts performance.

- Each planet is a `<mesh>` with a `<sphereGeometry>` (32-48 segments is enough for a hero-size
  sphere — don't go higher, it's wasted geometry at that viewing distance) and a
  `<meshStandardMaterial>`. You do NOT need real planet textures — flat/gradient colors with
  `roughness` and `metalness` tuned per "planet type" (rocky = high roughness, gas giant =
  lower roughness + a subtle emissive tint) read as intentional, not cheap.
- Give each planet an elliptical orbit path around a shared center by driving its position
  from `Math.cos(t * speed) * radiusX` / `Math.sin(t * speed) * radiusZ` inside `useFrame`,
  with a different `speed` and `radius` per planet so they don't move in visual lock-step.
  Also spin each planet on its own axis independently from its orbital motion.
  - If you want an accurate reference for orbit math and multi-planet choreography, the
    project at github.com/N3rson/Solar-System-3D demonstrates scaled orbital distances/speeds
    and is a good structural reference (do not copy its textures/assets, only the math pattern).
  - The npm package `react-planetary` (github.com/Bryce-Soghigian/react-planetary) ships
    ready-made planet components (Earth, Mars, Jupiter, etc.) if the user wants recognizable
    planets rather than abstract spheres — mention it as an option but default to the
    lightweight custom-sphere approach above unless the user asks for realism.
- Add one `<pointLight>` near the "core" of the scene so planets have visible shading/terminator
  lines instead of looking flat — this is what separates "planet" from "ball".

### Layer 3 — Meteoroids / asteroids

Small irregular rocks that streak or drift through the scene, adding motion at a different
scale than the slow galaxy rotation.

- Geometry: use a low-poly `<icosahedronGeometry args={[size, 0]} />` (detail level 0, NOT
  higher — the faceted look is what reads as "rock" rather than "sphere") and randomly perturb
  each vertex position slightly on creation for irregularity, OR just non-uniformly scale each
  instance on x/y/z for a cheaper irregular silhouette.
- Use `<Instances>` / `<Instance>` from `@react-three/drei` for meteoroids, not individual
  `<mesh>` components — this batches them into a single draw call. 15-30 meteoroids is plenty
  for a hero background; the reference project N3rson/Solar-System-3D uses thousands for a
  literal asteroid belt, which is overkill for a UI background and will hurt mobile performance.
- Animate each instance drifting along a slow, slightly randomized linear or curved path in
  `useFrame`, respawning at the opposite edge of the scene when it exits view (cheap infinite
  drift without ever growing the particle count).
- Color/material: dark grey-brown `meshStandardMaterial` with moderate roughness; keep them
  visually subordinate to the galaxy and planets — they're texture, not the focal point.

### Layer 4 — Scroll-driven camera zoom

The signature interaction: as the user scrolls, the camera flies toward the galaxy core, then
the scene hands off to normal page content.

- Drive this with GSAP `ScrollTrigger`, not `useFrame`/manual scroll listeners — ScrollTrigger
  gives you `start`/`end`/`scrub` for free and composes cleanly with the rest of a GSAP-based
  animation timeline elsewhere on the page.
- Reference pattern (seen in production R3F galaxy visualizations): track an `isZooming`
  boolean/progress value, animate the camera's position along a precomputed path toward
  `(0, 0, 0)` (the galaxy core) using an easing function — `power2.inOut` (GSAP) or a manual
  `easeInOutCubic` — never linear, linear camera movement reads as robotic. Continuously call
  `camera.lookAt(0, 0, 0)` during the animation so the core stays centered as you approach it.
- Bind the animation to `ScrollTrigger.create({ trigger: containerRef, start: 'top top', end:
  '+=100%', scrub: 1, onUpdate: self => { /* drive camera.position.z via self.progress */ } })`
  — `scrub: 1` ties the animation directly to scroll position with slight smoothing, which
  feels more intentional than a one-shot triggered animation.
- At the point where the zoom visually "reaches" the core (progress ~0.85-1), cross-fade the
  Canvas opacity to 0 while the real page content (Hero, nav, etc.) fades/scales in underneath
  — don't cut instantly, the dissolve is what sells the "flying through" illusion.
- Performance guard: set `frameloop="demand"` on the `<Canvas>` combined with `invalidate()`
  calls from the scroll handler, OR switch `frameloop` from `"always"` to `"demand"` once the
  zoom sequence completes and the scene is no longer visible, so idle GPU usage drops once the
  user has scrolled past the intro (this pattern is used in production R3F scroll experiences
  specifically to avoid burning GPU cycles on off-screen canvases).

## Performance checklist (always apply)

- One `BufferGeometry`/`Points` for the galaxy, not per-particle meshes.
- `<Instances>` for meteoroids, not individual `<mesh>` per rock.
- Cap `dpr` with `dpr={[1, 1.5]}` or similar on the `<Canvas>` — retina/high-DPI screens
  otherwise render at 2-3x the necessary resolution for a background element.
- Reduce or disable the effect entirely under `prefers-reduced-motion: reduce` — replace with
  a static, faded version of the same color palette rather than removing it outright, so the
  visual identity stays consistent for users with motion sensitivity.
- On narrow viewports (mobile), reduce particle count (e.g. 40-50% of desktop count) and
  planet/meteoroid counts rather than hiding layers outright — this preserves the aesthetic
  while protecting frame rate on weaker GPUs.
- Never use `THREE.Clock` (deprecated) — use the `clock` provided by `useFrame(state => ...)`
  or `state.clock.elapsedTime` instead.

## Color & mood guidance

The galaxy's "inside" and "outside" colors set the entire mood — treat this as the primary
creative lever, more than particle count or planet design:

- Warm core / cool arms (e.g. amber core fading to deep teal or indigo arms) reads as
  premium/editorial and pairs well with most modern UI palettes.
- Avoid defaulting to purple/violet-only galaxies — it's the most common choice and reads as
  generic "space UI" unless the rest of the brand palette is already purple-led. Ask what the
  site's existing accent colors are and derive the galaxy palette from those two colors instead
  of picking space-cliché colors independently.
- Keep meteoroid and planet materials desaturated relative to the galaxy particles — they
  should read as solid objects moving through a glowing field, not compete with the glow itself.

## Output expectations

When this skill is applied, produce:
1. A galaxy particle component (procedural, no external textures)
2. 2-4 planet components with independent orbit + spin animation
3. An instanced meteoroid field with drift/respawn behavior
4. A ScrollTrigger-driven camera rig that zooms toward the core and hands off to page content
5. Reduced-motion and mobile-density fallbacks for all of the above

Ask the user for their existing brand/accent colors before finalizing the galaxy palette if
they haven't already specified one — this single decision has the biggest visual impact on
whether the result feels bespoke versus template.
