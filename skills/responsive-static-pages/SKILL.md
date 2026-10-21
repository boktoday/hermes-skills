---
name: responsive-static-pages
description: "Use when building or editing static/here.now pages."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows, macos, linux]
metadata:
  hermes:
    tags: [html, responsive, mobile, here-now, static-site, verification]
---

# Responsive static pages (here.now and elsewhere)

Enforces a mobile baseline for any static HTML page (curriculum pages, hubs, brand pages, review pages, landing pages) and a Playwright overflow gate that must pass before publishing.

## When to Use

- Building or editing ANY HTML page that will be published to here.now.
- The user reports sideways scrolling, zoomed-out text, or elements cut off on a phone.
- As the LAST step before every publish of an HTML page: the verification gate below is mandatory.

## Baseline (apply while writing, not after)

1. **Viewport meta** on every page: `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`. Without it every rule below is moot.
2. **No un-wrappable horizontal rows.** A flex row of chips, steps, breadcrumbs or nav links MUST carry `flex-wrap: wrap` (plus `row-gap` for tidy lines). A no-wrap row wider than the phone stretches the ENTIRE page to its width. Symptom seen Oct 2026 on the curriculum TOC: the `COURSE 0 → … → COURSE 5` path strip pinned the whole page at 742px on a 390px phone (sideways scroll, "padding is off" report).
3. **Grids must shrink:** `grid-template-columns: repeat(auto-fit, minmax(min(320px, 100%), 1fr))`. Bare `minmax(320px, 1fr)` overflows any container narrower than 320px (any 360px phone after page padding).
4. **Long strings:** `overflow-wrap: anywhere` on text containers (URLs, code, names, emails).
5. **Images/media:** `img { max-width: 100%; height: auto; }`; fixed-width carousels use `flex: 0 0 min(240px, 82vw)`, never percentage grid columns.
6. **Tap targets** on interactive elements: at least 44x44 on touch-sized screens.
7. **Fluid type:** headings use `clamp()`; never a fixed px page width; wrap padding stays 20-28px on phones.
8. **Mobile navigation = a working hamburger (standing rule, 14 Oct 2026).** Applies to EVERY website built for the owner, not just here.now. If nav links cannot fit, never leave them as `display:none` with nothing in their place: hide the desktop links behind a real hamburger button (44px+ tap target, `aria-expanded`, stacked drop-down panel, close on link click, burger morphs to X). Watch the between-breakpoints zone: a `space-between` header with `flex-wrap` silently wraps the burger onto a second row at ~390px when the wordmark can't shrink — give the wordmark `flex:1 1 auto; min-width:0` and the header `flex-wrap:nowrap` on mobile.

## The verification gate (required before publish)

Run `scripts/responsive_check.py` against the local file (or the live URL). PASS requires `document.documentElement.scrollWidth == clientWidth` at EVERY tested width (360, 390, 768, 1280) and an empty offender list (elements whose right edge exceeds the viewport), plus one full-page screenshot at 360 for the eyeball pass.

```
cd <skill>/scripts
PLAYWRIGHT_BROWSERS_PATH="<YOUR_HOME>/AppData/Local/ms-playwright" \
  "<HERMES_HOME>/hermes-agent/venv/Scripts/python.exe" \
  responsive_check.py <url-or-file> [more...]
```

Never publish while the 360px check fails. A page that overflows at 360 is a sideways-scrolling page in the user's hand.

## Fix recipes

- **Rigid chip/step row** → add `flex-wrap: wrap` (e.g. `gap: .6rem 1rem; flex-wrap: wrap;`). One-line fix, whole-page effect.
- **Grid overflow** → wrap the min: `minmax(min(320px, 100%), 1fr)`.
- **Table wider than phone** → wrap in a container with `overflow-x: auto`.
- **Off-canvas panel peeking** from the edge → usually an open/animating menu, not a bug; verify with `elementFromPoint` before touching it. If genuinely stuck, ensure the closed state is `display: none` or fully translated out with the parent clipping.

## Vertical rhythm audit (spacing, not overflow)

The gate above only catches sideways breakage. Text pressed against the block below, or a hole where a margin went missing, passes every numeric check and still reads as broken on a phone. Measure it:

```python
# inside a Playwright page at 390 and 1440
data = pg.evaluate("""(() => {
  const out = [];
  document.querySelectorAll('.wrap').forEach(w => {
    const kids = [...w.children].filter(k => k.textContent.trim());
    for (let i = 1; i < kids.length; i++) {
      const a = kids[i-1].getBoundingClientRect(), b = kids[i].getBoundingClientRect();
      out.push({a: kids[i-1].tagName + '.' + kids[i-1].className,
                b: kids[i].tagName + '.' + kids[i].className,
                gap: Math.round(b.top - a.bottom)});
    }
  });
  return out;
})()""")
print(sorted([d for d in data if d['gap'] >= 0], key=lambda d: d['gap'])[:8])
```

Rules: **ignore negative gaps** (grid and flex columns overlap vertically by definition); anything **under 14px** is text touching the next block, anything **over 90px** is a missing or doubled margin. Fix with explicit margins on the specific block (`#course .weeks{margin-top:28px}` / `22px` mobile) rather than nudging a global section padding, which moves everything.

- **Sticky headers eat anchor targets.** Any page with a sticky header needs `section[id]{scroll-margin-top:84px}` or `#apply`, `#work` and friends land with the heading hidden behind the bar. Verify by clicking the anchor, not by reading CSS.
- **A gap of exactly 0 between a paragraph and the grid below** is the most common finding: an inline `margin:18px 0 0` on the paragraph plus `margin-top:0` on the grid. It looks fine in a tall desktop screenshot and cramped on a phone.

## Cross-page sweep

Sibling pages usually carry their own COPY of the shared CSS block. Fix the source template AND grep the published siblings for the offending selector, fixing each. On here.now: replace each changed file in place (etag-guarded), ONE publish, then re-point the mount (see the `here-now` skill). Verify the live page, not just the local file: CDN propagation can take a minute; poll for a distinctive marker string.

## Pitfalls

- **Never trust the desktop screenshot.** Overflow bugs only show at narrow widths. Always render 360px.
- **Overflow numbers are NOT a visual double-check (user correction, 14 Oct 2026: "you need to double check your work").** The numeric gate passes a page whose nav simply vanishes on mobile, and passes a menu button whose white text was silently overridden by a `.menu a { color: ... }` rule into ink-on-ink invisibility. After the gate: screenshot the mobile width in BOTH states (menu closed AND open) and actually LOOK at them before delivering. Check: burger on the same row as the wordmark, menu panel readable, every button's label visible against its own background, headings not breaking mid-compound (wrap gradient phrases in `white-space:nowrap` spans at word boundaries).
- **`overflow-x: hidden` hides the bug, it does not fix it** (content becomes unreachable). Fix the offending element first; at most use it as belt-and-braces after the gate passes.
- **Offender scans need a width filter** (`r.width > 8`) or every text span reports as an offender, and elements inside an intentional inner scroller (`overflow-x: auto` ancestor) must be exempt — otherwise a by-design scroll region false-fails the gate.
- **A flex or grid item wrapping a `min-width` SVG (or wide table) never shrinks below it** — the whole page overflows even though the inner container has `overflow-x: auto`. Give the item `min-width: 0; max-width: 100%` and keep the scrolling on the inner wrapper. (Caught by the gate on the BWJ diagram pages, Oct 2026.)
- **The white panel at a phone screen edge in a user's screenshot is usually their device overlay** (edge-panel handle) or the site's own menu mid-animation. Reproduce before assuming site code is at fault; check the live page with `elementFromPoint`.
