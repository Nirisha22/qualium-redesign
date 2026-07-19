# Qaulium AI — UI/UX Redesign

A complete visual redesign of [qauliumai.in](https://qauliumai.in). **Presentation layer only** — every section, navigation item, route, form field, and piece of copy from the live site is preserved. The information architecture is unchanged.

> This is an unaffiliated redesign concept built against the public site. It is not deployed and does not represent Qaulium AI.

## Running it

```bash
npm install
npm run dev        # http://localhost:3000
npm run verify     # typecheck + lint + tests + build — run this before shipping
npm test           # 18 tests over the quantum simulator
npm run typecheck
npm run lint
npm run build
```

Node 20.9+ required by Next 16; the test suite needs Node 22.6+ for native
TypeScript stripping (no build step, no test-runner dependency).

## Stack

Next.js 16 (App Router, Turbopack) · React 19 · TypeScript · Tailwind CSS v4 · Framer Motion · React Three Fiber + drei + three · Lenis · GSAP

Two deviations from the original brief, both deliberate:

- **Next 16, not 15.** `create-next-app@latest` now ships 16; downgrading to satisfy a spec written before 16 existed made no sense. Async `params`/`searchParams` are handled accordingly.
- **No shadcn/ui.** Every component here is bespoke glass-and-light. shadcn's Radix-based primitives would have been dead weight — nothing in this design maps onto its defaults.

## Structure

```
src/
├─ app/                    routes; one file per page, metadata co-located
│  ├─ software/[slug]/     four product pages from one data-driven template
│  ├─ careers/apply/       reads ?role= to preselect the application form
│  ├─ sitemap.ts robots.ts
│  └─ not-found.tsx
├─ components/
│  ├─ layout/              Navbar, Footer, Wordmark, SmoothScroll (Lenis)
│  ├─ sections/            page sections; Specimens.tsx holds the six field-guide visuals
│  ├─ three/               R3F scenes, lazy-mounted and code-split
│  └─ ui/                  GlassCard, MagneticButton, EnergyTimeline, Form, Ambient, Reveal
├─ data/                   ALL copy lives here, verbatim from the live site
└─ lib/                    cn(), shared motion tokens
```

**Content is data, not markup.** Every string sits in `src/data/*.ts`. Editing copy never means touching a component.

## Design system

Defined once in `globals.css` as Tailwind v4 `@theme` tokens.

| | |
|---|---|
| Surfaces | `void #04050a` → `abyss` → `surface` → `elevated` |
| Ink | `ink` → `ink-soft` → `ink-muted` → `ink-faint` |
| Light | `violet #7c5cff`, `azure #3b82f6`, `cyan #22d3ee` |
| Easing | `cubic-bezier(0.16, 1, 0.3, 1)` — long, slow-out |

Custom utilities: `glass`, `shell`, `display`, `eyebrow`, `text-lumen`, `gradient-ring`.

## The circuit sandbox

`/software/studio#sandbox` runs a genuine two-qubit statevector simulator — `src/lib/quantum.ts`, no dependencies, ~250 lines of exact complex linear algebra. Build a circuit from H/X/Y/Z/S/T/CNOT, scrub through it gate by gate, and watch amplitudes, probabilities and two 3D Bloch spheres update from the same state. "Run 1,024 shots" samples the real distribution.

It lives on the Studio page rather than the homepage on purpose: the brief said not to invent homepage sections or alter the IA, and Studio *is* the visual circuit builder. It demonstrates four products at once — builder, simulator, debugger (the step-through slider), and Bloch visualisation.

The physics is the point, so it's tested rather than eyeballed. Verified properties include:

- gate correctness (`X` on q0 → `|10⟩`, qubit 0 being the most significant bit)
- `H` → `+X` on the Bloch sphere, `H·S` → `+Y`
- **entanglement is visible**: after H+CNOT both Bloch vectors collapse to the origin, because each qubit's reduced density matrix is maximally mixed. `|r| < 1` is what drives the "entangled" badge — it isn't hardcoded.
- norm preserved across every preset; the self-inverse preset returns exactly to `|00⟩`
- sampling: exact totals, zero counts for zero-amplitude outcomes, frequencies converging to |ψ|²

Tests live in `src/lib/quantum.test.ts` and run via `npm test` — `node --test` with native TypeScript stripping, so there's no test-runner dependency and no build step. That's why `allowImportingTsExtensions` is on in `tsconfig.json`: Node requires the explicit `.ts` extension on relative imports.

## Command palette

⌘K / Ctrl-K, with a visible affordance in the navbar for discoverability. It indexes nav, products, research posts, job roles and footer links — all existing destinations. A navigation aid, not new IA.

## Explanatory visuals

The site shows what the platform does rather than only describing it. Three families of visual, all server-rendered and all frozen under `prefers-reduced-motion`:

- **`sections/CardDemos.tsx`** — a looping demonstration on each platform card: a gate dropped on a wire while the code writes itself (bi-directional sync), shots converging into a distribution against a hardware twin, a prompt resolving into gates, a playhead scrubbing through execution frames.
- **`sections/UseCaseArt.tsx`** — the actual computation behind each domain: a ligand docking into a receptor pocket, Monte Carlo paths fanning from a spot price, circulation bands crossing a gridded globe, a lattice with its shortest vector pulsing.
- **`sections/Specimens.tsx`** — the six field-guide instruments.
- **`sections/NirvanaArt.tsx`** — one diagram per hardware subsystem, drawing the actual mechanism: light routed through Mach-Zehnder switches, a coherent state squeezed in phase space (area conserved, shape traded), a photon parked in a fibre delay loop via `offset-path`, and modules locked by a sync pulse.
- **`sections/PostCover.tsx`** — generative cover art per research post, keyed off its subject: a hybrid network for the QML piece, five modalities for the "five ways to build a qubit" piece, diverging growth curves for classical-vs-quantum.

**Why SVG and not more 3D:** each WebGL context costs real memory and browsers cap them near 16. Twenty animated diagrams as SVG cost approximately nothing and stay crisp at any zoom. 3D is reserved for the four places where volume and lighting genuinely carry meaning — the hero core, the stack platform, inner-page hero objects, and the Bloch spheres.

## Notable implementation details

**3D is lazy and optional.** `useLazyMount` defers every WebGL scene until it's near the viewport and the browser is idle, behind a `next/dynamic({ssr:false})` chunk. A CSS approximation of the hero core renders first and stays permanently on devices without WebGL. Three details there are load-bearing and easy to regress:

- `requestIdleCallback` must stay bound to `window` or it throws `Illegal invocation`.
- It must carry a `timeout`. This page animates continuously, so the main thread never reports true idle and a timeout-less callback never fires.
- IntersectionObserver callbacks don't arrive in some embedded browsers, so a failsafe timer backs it up.

**Lighting is hand-built, not an HDRI.** `Scene.tsx` uses drei `<Lightformer>`s rather than `<Environment preset>`, which would stream an `.hdr` from a CDN — an unwanted dependency under a strict CSP or offline.

**Deterministic starfield.** Positions come from a seeded hash, and every value is `toFixed(3)` before reaching a style object. Full-precision floats serialise differently server- vs client-side, and the resulting attribute mismatch aborts hydration for the entire tree.

**The 3D stack never owns the interaction.** `Stack.tsx` keeps an accessible button list as the only control surface; the WebGL platform beside it purely reflects that state. Losing WebGL costs the spectacle, never the ability to use the section. Scroll position reaches the scene through a ref, so scrolling never triggers a React render.

**Motion respects the user.** `prefers-reduced-motion` disables Lenis outright, collapses reveals to plain fades, freezes every looping demo into a readable static diagram, and slows or stops all 3D animation.

**Page transitions** live in `app/template.tsx`, which (unlike `layout.tsx`) remounts per navigation.

## Accessibility

Skip-to-content link · single `<h1>` per page with ordered headings · visible focus rings throughout · `aria-expanded`/`aria-haspopup` on nav triggers, Escape to close · all decorative canvases and ambient layers `aria-hidden` · labelled form controls with `aria-live` status on submit · the 3D stack reflects state but never owns the interaction · touch targets ≥44px.

**Contrast is measured, not assumed.** Every ink token clears WCAG AA (4.5:1) for body text against the darkest surface it can appear on (`elevated`, the hovered-card fill):

| token | vs `void` | vs `elevated` |
|---|---|---|
| `ink` | 18.8 | 15.7 |
| `ink-soft` | 12.3 | 10.3 |
| `ink-muted` | 6.8 | 5.7 |
| `ink-faint` | 5.5 | 4.6 |
| `violet-text` | 7.5 | 6.3 |

Note the split between `violet` (`#7c5cff`) and `violet-text` (`#a78bfa`). The vivid violet is decorative only — borders, glows, gradients — where the 3:1 UI-component threshold applies. It fails AA as body text on glass (4.37:1), so **never use `text-violet`**; use `text-violet-text`.

## Performance

Three WebGL scenes can exist on a page, so each one is gated:

1. **Not mounted** until near the viewport and the browser is idle (`useLazyMount`), behind a `next/dynamic({ssr:false})` chunk.
2. **Not rendering** when scrolled off-screen or the tab is hidden — `useVisibility` drives R3F's `frameloop`, dropping an idle scene to zero GPU work.

`useVisibility` deliberately **fails open**: it defaults to visible and only pauses on an explicit "not intersecting" report, so a browser that never fires IntersectionObserver keeps animating rather than freezing on a blank canvas.

## Deploying

**Before anything else:** this is a redesign of a site owned by someone else. Publishing it under the Qaulium AI name, branding, or domain would impersonate a real company. Ship it as a clearly-labelled concept, or with the branding replaced — not as them.

### Required

1. **`NEXT_PUBLIC_SITE_URL`** — set to the deployment origin, no trailing slash (see `.env.example`). It drives canonical tags, OG URLs and `sitemap.xml`. If unset it falls back to `http://localhost:3000`, which would emit canonicals pointing at localhost.
2. **Wire the forms.** `GlassForm` currently intercepts submit and shows a success state locally. Registration, contact and job applications all silently discard input until it's connected to a Server Action or route handler. This is the single biggest functional gap.
3. **Write `/privacy` and `/terms`.** They intentionally contain no legal text.
4. **Security headers need a Node runtime.** `next.config.ts` sets CSP, HSTS, `X-Frame-Options`, `Referrer-Policy` and `Permissions-Policy` via `headers()`. On a static host these are ignored — configure them at the CDN or edge instead.

### Measured

| | |
|---|---|
| Initial JS (brotli) | **243 KB** |
| Initial CSS (brotli) | 10 KB |
| **Total first load** | **253 KB** |
| three.js + drei chunk | 207 KB brotli — **deferred**, fetched only when a scene mounts |
| Routes | 19, all prerendered except `/careers/apply` (reads `?role=`) |

CSP is verified to produce **zero violations** with WebGL, blob canvases and the inline SVG grain all working. `'unsafe-eval'` is dev-only (Turbopack HMR); production drops it.

### Verified

`npm run verify` (typecheck + lint + 18 tests + build) · all 19 routes return 200, unknown paths 404 · no horizontal overflow at 375px on the homepage, sandbox and the 14-field form · no tap target under 44px · contrast measured against WCAG AA · `x-powered-by` disabled · default Next favicon and starter assets removed.

## Known gaps

- **Forms have no backend.** Submission is intercepted client-side and shows a success state. Wire `GlassForm` to a Server Action or route handler to make it live.
- **`/privacy` and `/terms` are intentionally empty.** The live site links both to `#`, so there was no source document. Rather than invent legal text — harmful if anyone relied on it — these routes state the document is pending and point to `support@qauliumai.in`. Replace with counsel-reviewed copy before any real launch.
- **Research posts don't link to individual articles.** The live site's cards link only to Medium; these do the same.
- **Only the root OG image is custom.** `app/opengraph-image.tsx` covers every route; per-route cards would need one file per segment.
- **Never edit source files with PowerShell's `Get-Content -Raw` + `Set-Content`.** On this repo it reads UTF-8 as ANSI and writes it back mangled, turning `—` into `â€"` and `⟩` into `âŸ©` across every file it touches. Use the editor, or Node with explicit `utf8` encoding.
- `SITE.url` in `src/data/site.ts` points at the production domain for canonicals and sitemap. Change it before deploying anywhere else.

## Fixes applied to live-site bugs

- The navbar's "Get Early Access" CTA is `href="#"` on the live site (dead). It points to `/registration` here.
- The live `/research` page links "Follow on Medium" to `medium.com/@cheif.research`, which contradicts the site-wide footer's `medium.com/@qauliumai`. The footer's account is used throughout.
- The registration modal auto-opens on page load on the live site. Registration is a destination here (`/registration`), reached by CTA rather than interrupting the visitor.
