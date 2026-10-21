---
name: web-app-debugging
description: "Debug HTML/JS web app issues: script boundaries, function scope, inline JS pitfalls, and deployment-related breakage."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows, macos, linux]
tags: [debugging, "web", html, javascript, deployment]
metadata:
  hermes:
    tags: [debugging, "web", html, javascript, deployment]
    related_skills: [systematic-debugging, dogfood]
triggers:
  - "Browser tool hits an error"
  - "JS function undefined at runtime"
  - "Web Share API cancelled"
  - "Multi-line string syntax error"
  - "Screen changes to wrong content after toggle/click"
---

# Web App Debugging

## A DOM update that silently does not happen: probe the DOM call

Symptom: a component renders, but every state update leaves the DOM unchanged. No readable exception
reaches `window.onerror`, or you get a generic `insertBefore` TypeError from inside a framework's
minified diff, pointing at internals you cannot read.

**Patch the DOM method itself to catch the offending argument.** This names the exact subtree and the
node it tried to insert before, in one run, where the minified stack does not:

```js
(function(){
  var _ib = Node.prototype.insertBefore;
  Node.prototype.insertBefore = function(n, ref){
    if (!(n instanceof Node)) {
      window.__bad = window.__bad || [];
      window.__bad.push({
        kind: typeof n, str: String(n),
        parent: this.nodeName,
        parentHTML: (this.outerHTML || '').slice(0, 200),
        children: Array.from(this.childNodes, c => c.nodeName),
        ref: ref ? ref.nodeName : null,
        stack: (new Error().stack || '').split('\n').slice(1, 7).join(' | ')
      });
    }
    return _ib.call(this, n, ref);
  };
})();
```

Inject it before the app's module, clear `window.__bad` immediately before the interaction under test,
then read it back. `parentHTML` plus `children` show what the DOM actually looks like at the moment of
failure, which is usually where the answer is. A parent whose first child is a stray text node holding
a tag name (`"div"`) means the tree was spliced, not that a value was wrong.

## Verify a state update landed, do not assume it

An app can render correctly and still never update. Assert on the DOM after the interaction: re-read an
input's value, check a dependent element's text changed, check a `disabled` attribute flipped. "The
click succeeded" and "the handler ran" are both compatible with a render that threw.

**Drive inputs the way a person does.** `fill()` sets the value and fires one `input` event; a listener
wired to key or composition events may never see it, so a genuine failure looks like a passing test.

## Children that evaluate to an empty string

`${cond && vnode}` yields `''` when `cond` is `''`, and `${cond ? x : ''}` yields `''` by design. An
empty-string child leaves a hole in the child list that the next diff cannot insert against. Use `null`
for "nothing", never `''`, and audit before shipping:

```python
re.findall(r"\$\{[^\n]*\?[^\n]*: ''\}", html)   # ternary falling back to an empty string
re.findall(r"\$\{[^\n]*&& html", html)           # && chain that can yield ''
```

## A component must return the same root element every render

Returning `<main>` for a loading state and `<div>` once loaded changes the root element type between
renders and leaves the renderer's view of the DOM out of step. Render the shell unconditionally and put
the loading state inside it. The same applies to any early return that changes the wrapper.

## When to stop debugging a renderer and drop it

A local-first single-file app has no dependency that cannot be removed. If a renderer fault survives a
bisect and a DOM probe, and the app's own logic is already correct, rewriting the draw layer in plain
DOM is usually faster and strictly better for a teaching artefact: no CDN, no version drift, works
offline, and the source stays readable. Do not spend an unbounded budget on a library that was only
there to be convenient. Report honestly that the fault was isolated but not root-caused.

## Verify what actually rendered, not what the CSS says

A computed style is not proof of a pixel. When an element looks wrong on screen, do not stop at `getComputedStyle` returning the value you expected, and do not trust the visual impression alone. Two tools settle it.

**Rasterise the element's own SVG in isolation.** Serialise the live node to a data URI, draw it to a canvas, and read the pixels. If the isolated render is correct, the markup and the `currentColor` chain are fine and the fault is in compositing, stacking, or an overlay:

```javascript
const svg = document.querySelector('.plate svg');
const xml = new XMLSerializer().serializeToString(svg);
const img = new Image();
img.src = 'data:image/svg+xml;base64,' + btoa(unescape(encodeURIComponent(xml)));
await new Promise((ok, no) => { img.onload = ok; img.onerror = no; img.src = img.src; });
const c = document.createElement('canvas'); c.width = c.height = 400;
const ctx = c.getContext('2d'); ctx.drawImage(img, 0, 0, 400, 400);
const d = ctx.getImageData(0, 0, 400, 400).data;
let darkest = [255,255,255];
for (let i = 0; i < d.length; i += 4)
  if ((d[i]+d[i+1]+d[i+2])/3 < (darkest[0]+darkest[1]+darkest[2])/3) darkest = [d[i],d[i+1],d[i+2]];
```

**Then sample the composited screenshot.** Load the saved PNG and count real colours in the element's bounding box, against the exact RGB the CSS declares. A wash-out that survives an isolated black render is a compositing problem:

```python
im = Image.open(shot).convert("RGB"); px = im.load()
target = (245, 245, 241)                       # the declared --paper-2
hits = [(x, y) for y in range(y0, y1, 2) for x in range(x0, x1, 2) if px[x, y] == target]
print("declared colour present:", len(hits), "of", ((y1-y0)//2)*((x1-x0)//2))
```

If the count is zero, something between the element and the screen is lightening it. Check `elementsFromPoint` at the element's centre to see the full paint stack, and read `filter`, `mixBlendMode`, and `opacity` on the element and every ancestor — an ancestor with a `mask-image`, `filter`, or `opacity` creates a compositing layer whose blend can wash out descendants even when every computed value on the child is correct. That is a real mechanism, not a guess: report it as the suspect and keep probing rather than declaring the CSS wrong.

Report an unfixed rendering fault honestly, with the evidence gathered. "Computed values are correct, the isolated SVG renders pure black, the composited pixels are 254/254/254, the paint stack shows no overlay" is a genuinely useful finding; "it looks washed out" is not.

**Overlays on illustrated art collide in rendered space, not in source coordinates.** A badge positioned over a group that carries a rotation (`rotate(-2)`) lands on a different chip than the unrotated numbers suggest (a FREE badge covering a PRINTED chip), and text sized flush to its rect overflows once the real font renders wider than estimated. After placing any free-positioned chip or callout on illustrated/rotated art, element-screenshot the artwork at 1.5x and inspect the seams; fix by repositioning in rendered space, not by shrinking type.

## An image that fetches fine but never renders

An HTTP 200 proves the bytes exist, not that the browser painted them. When a user reports broken
images, separate three failure states before touching any file:

1. **Never requested** — `complete=false, naturalWidth=0`, and no network request for the URL. The
   usual cause is `loading="lazy"` on an image that never crossed the load threshold (below-fold
   photos on a page the viewer did not scroll just right read as permanently broken). This is a
   markup finding: for a site with a handful of small images, drop `lazy` and load eagerly.
2. **Request failed** — an actual network error (404, abort, TLS); the request log names it.
3. **Corrupt bytes** — 200 with a plausible size that cannot decode: validate structurally
   (`PIL.Image.verify()` / magic bytes) and hash-compare against the local master; a truncated or
   mangled upload sails past size checks and shows as a broken icon.

Probe every candidate in a real browser — `Array.from(document.images).map(i => i.complete &&
i.naturalWidth > 0)` — and bring each image into view explicitly (`el.scrollIntoView(); await
el.decode()`) before judging it: a fast programmatic scroll can miss lazy-load triggers, making a
healthy image look dead. Distinguish never-requested from failed-request before blaming anything
server-side.

## Drive a local static build over plain HTTP before screenshotting

Serve the build directory and open it over `http://127.0.0.1:PORT` rather than `file://` when the page fetches relative assets, a stylesheet, or a font. It matches how it will be served, avoids `file://` origin quirks, and makes a 404 immediately visible in the network panel instead of silently dropping a stylesheet.

```bash
cd /path/to/build && python3 -m http.server 8899 --bind 127.0.0.1
```

## Set the viewport explicitly and re-set it after every capture

`Emulation.setDeviceMetricsOverride` persists across navigations. A capture taken at phone width and then examined at desktop width is a misleading measurement, and a stale override silently changes what the next call reports. Set the metrics you want in the same call as the navigation, and reset the device scale factor to 1 when you change it back — a `deviceScaleFactor: 2` clip-capture frequently times out while a 1x full capture succeeds, so do not conclude the page is heavy from a 2x timeout.

**`capture_screenshot` timing out is not evidence the page is heavy.** It is a harness limit. Retry at `deviceScaleFactor: 1` and, if the element is off-screen, scroll it into view first rather than widening the viewport to the document height (which changes layout for any `vh`/`aspect-ratio` rule, per the long-page trap above).

## Overview

Techniques for debugging client-side HTML/JS web apps — especially static or single-page apps deployed via CDN, where the browser's dev tools and HTML boundaries are the primary diagnostic tools.

Use this skill when:
- A web app feature silently does nothing (no error, no response)
- JS functions exist in the source but are `undefined` at runtime
- Buttons/links have `onclick` handlers that don't fire
- The app works locally but breaks when deployed
- Browser console shows no JS errors but features are broken

## Core Principle

**What the browser renders is what executes.** The HTML file served to the browser defines the entire execution environment. If code isn't inside a valid `<script></script>` tag, it's display text — never executed, never noticed by the developer who's looking at the right source file and seeing the right code.

## Diagnostic Toolkit

### 1. Count and Inspect Script Tags

```javascript
// How many script tags are executing?
document.querySelectorAll('script').length

// Check each one's size
[...document.querySelectorAll('script')].map(s => s.text.length)

// First 100 chars of each script
[...document.querySelectorAll('script')].map(s => s.text.substring(0, 100))
```

Expected: every function the app needs should be inside one of these script tags. If a function exists in the raw HTML but doesn't appear in any script's `.text`, it's orphaned.

### 2. Check Function Existence

```javascript
typeof exportData          // → "undefined" = not defined in any script
typeof showHelp             // → "undefined"
typeof toggleDarkMode       // → "undefined"
```

If the HTML source shows `function exportData(){...}` but `typeof` is `undefined`, the function is not inside a `<script>` tag.

### 3. Locate Orphaned Code

```javascript
// Find the last script boundary
var html = document.documentElement.outerHTML;
var lastClose = html.lastIndexOf('</script>');
var afterScript = html.substring(lastClose + 9);  // +9 for "</script>"

// Does it contain JS functions?
var nextOpen = afterScript.indexOf('<script>');
if (nextOpen === -1 && afterScript.indexOf('function ') >= 0) {
  console.log('ORPHAN JS FOUND after last </script>');
}

// Check for help overlay / HTML content separating JS from scripts
var helpPos = afterScript.indexOf('<div id="helpOverlay"');
if (helpPos > 0) {
  var orphanCode = afterScript.substring(0, helpPos);
  console.log('Orphan JS before help overlay:', orphanCode.length, 'chars');
}
```

### 4. Verify via curl/Grep

```bash
# Count script tags in the raw HTML
curl -s [URL] | grep -c '<script>'
curl -s [URL] | grep -c '</script>'

# If counts don't match, there's a problem
# Check for orphaned code after the last </script>
curl -s [URL] | grep -A 5 '</script>' | tail -5
```

**Why this matters for landing pages:** responsive rules almost always live in the CSS file because that's where the breakpoints and `clamp()` math belong. But if a key element's HTML template hard-codes the size, every media query in the stylesheet becomes silently inert for that element. Easy to miss because DevTools shows the desktop size and you assume the rule "isn't working" — when in fact the rule is working fine, you just have a competing inline style winning.

### Pattern 10: `patch` Tool Whitespace Drift on HTML Files

**Symptom:** Edit one part of an HTML file using `patch(old_string, new_string)`. After publishing, the live page is *visually identical* but inspection reveals mismatched indentation — new `</div></section>` closers sit at the wrong column, or unclosed wrapping divs leak into adjacent sections.

**Root cause:** `patch` matches `old_string` exactly character-for-character. When your `new_string` has slightly different leading whitespace (say one extra indent level because you copied from a different chunk), the substitution lands clean but the surrounding context's indentation is now wrong. Worse, if your `old_string` includes the *closing* tags of a wrapper, the matching closes get pushed out of alignment after the swap.

**Three reliable fixes (in order of preference):**

1. **Use `write_file` for single-purpose edits.** Read the full file, edit in memory, write the whole thing back. You control indentation across the full file, not just the patch region. This is the only fully reliable fix.

2. **Use a Python helper script** (when `write_file` is blocked or you want atomicity): read the file, do an in-Python string replace on the exact bytes, write back. Runs from `terminal`, not `execute_code`. Survives `cron_mode` blocks.

3. **For targeted `patch` ops, include the wrapper opening AND closing in `old_string`** — don't just patch the middle. That way both sides get rewritten with consistent indentation:
   ```python
   # BAD — only includes the inside, leaves closing tags misaligned
   old_string = '<div class="cta-row" style="...">\n  button\n</div>'
   
   # GOOD — includes the wrapper opens AND the closes
   old_string = '''<section>\n  <div class="cta-row">\n    button
  </div>
</section>'''
   ```

**Detection:** Run `grep -c` for the closing tag count BEFORE and AFTER the patch. If the count changes unexpectedly, the patch broke the structure. Also: `python -c "import html.parser; p=html.parser.HTMLParser(); p.feed(open('file.html').read())"` — if the parser raises, the file is malformed (works offline).

**Recovery order:** `patch` is not transactional. If a patch drifts, the next patch needs to match the (now-broken) file exactly. Don't try to "fix the fix" in a separate patch — read the file in full, see the actual state, then re-patch from the current state.

### Pattern 11: Inline `style="..."` Attributes Override CSS Media Queries (Even With `!important`)

**Symptom:** Hero/section text is too small on a small phone, even though the CSS file has `@media (max-width: 480px) { .hero-time { font-size: clamp(...); line-height: ... } }` with `!important`. The media-query rule exists, the breakpoint matches, but the element still renders at the desktop size. Inspecting the element in DevTools shows the inline style is winning.

**Root cause:** Inline `style` attributes (e.g. `<div class="hero-time" style="font-size: clamp(2.8rem, 6.5vw, 4.8rem)">`) are presentational at the highest specificity — they beat even `!important` rules in external CSS for the properties they set. Media-query rules with `!important` only win if no inline `style` exists for that same property.

**Common source:** the markup was generated from a template or earlier hand-edit that hard-coded the desktop size as an inline `style`. The CSS later got responsive rules added, but nobody removed the inline override.

**Fix:**

1. **Remove the inline `style` from the markup.** Let the CSS class handle the property for all viewports:
   ```html
   <!-- BEFORE -->
   <div class="hero-time" style="font-size: clamp(2.8rem, 6.5vw, 4.8rem)">
   <!-- AFTER -->
   <div class="hero-time">
   ```
   The class rule (`.hero-time { font-size: clamp(2.8rem, 6.5vw, 4.8rem) }` plus `.hero-time { font-size: clamp(1.4rem, 5vw, 1.75rem) !important }` inside `@media (max-width: 480px)`) now wins on mobile because there's no inline override to fight.

2. **For inline SVGs / dynamic-width content** (logos, brand marks), keep the inline `style` but make it responsive: `style="width: min(NNNpx, 90%); max-width: 100%; height: auto"`. The `min(400px, 90%)` syntax means "as big as 400px, but never wider than 90% of the container" — gives you a desktop cap and a mobile floor in one declaration.

**Detection:**
- View-source on the page, search for the property name (e.g. `font-size:`) and look for inline `style=` instances on the affected element.
- In DevTools, switch the Computed tab; if the inline style value matches the desktop declaration and the CSS rule shows `!important`, you have an inline-style-vs-CSS fight.
- Quick audit before deploy: `grep -E 'style="[^"]*font-size' file.html` — any hits on responsive-only elements are a smell.

**Why this matters for landing pages:** responsive rules almost always live in the CSS file because that's where the breakpoints and `clamp()` math belong. But if a key element's HTML template hard-codes the size, every media query in the stylesheet becomes silently inert for that element. Easy to miss because DevTools shows the desktop size and you assume the rule "isn't working" — when in fact the rule is working fine, you just have a competing inline style winning.

### Pattern 10: `patch` Tool Whitespace Drift on HTML Files

**Symptom:** Edit one part of an HTML file using `patch(old_string, new_string)`. After publishing, the live page is *visually identical* but inspection reveals mismatched indentation — new `</div></section>` closers sit at the wrong column, or unclosed wrapping divs leak into adjacent sections.

**Root cause:** `patch` matches `old_string` exactly character-for-character. When your `new_string` has slightly different leading whitespace (say one extra indent level because you copied from a different chunk), the substitution lands clean but the surrounding context's indentation is now wrong. Worse, if your `old_string` includes the *closing* tags of a wrapper, the matching closes get pushed out of alignment after the swap.

**Three reliable fixes (in order of preference):**

1. **Use `write_file` for single-purpose edits.** Read the full file, edit in memory, write the whole thing back. You control indentation across the full file, not just the patch region. This is the only fully reliable fix.

2. **Use a Python helper script** (when `write_file` is blocked or you want atomicity): read the file, do an in-Python string replace on the exact bytes, write back. Runs from `terminal`, not `execute_code`. Survives `cron_mode` blocks.

3. **For targeted `patch` ops, include the wrapper opening AND closing in `old_string`** — don't just patch the middle. That way both sides get rewritten with consistent indentation:
   ```python
   # BAD — only includes the inside, leaves closing tags misaligned
   old_string = '<div class="cta-row" style="...">\n  button\n</div>'
   
   # GOOD — includes the wrapper opens AND the closes
   old_string = '''<section>\n  <div class="cta-row">\n    button
   </div>
</section>'''
   ```

**Detection:** Run `grep -c` for the closing tag count BEFORE and AFTER the patch. If the count changes unexpectedly, the patch broke the structure. Also: `python -c "import html.parser; p=html.parser.HTMLParser(); p.feed(open('file.html').read())"` — if the parser raises, the file is malformed (works offline).

**Recovery order:** `patch` is not transactional. If a patch drifts, the next patch needs to match the (now-broken) file exactly. Don't try to "fix the fix" in a separate patch — read the file in full, see the actual state, then re-patch from the current state.

**Variant: patch tool "path required" silent error on multi-line edits.** When calling the `patch` tool with `old_string`/`new_string` containing literal newlines, the tool can reject the call with `"path required"` even when a `path` parameter is provided. This appears to be a parameter-parsing failure on multi-line strings that contain certain characters (backslashes, double-quotes mixed with single-quotes, leading whitespace, embedded closing tags). Workaround: write the new text to a `.py` helper file with `write_file` and execute it from `terminal` via `python helper.py`. The `write_file` path is unaffected by the same parsing quirk.

### Pattern 12: An Override Loses to Specificity, Not to Source Order

**Symptom:** You append a stylesheet at the end of the file with a plain class override (`@media (min-width: 901px) { .cover-grid { grid-template-columns: 1.2fr 1fr } }`) and nothing changes. Moving it later in the file, re-rendering, and cache-busting do nothing.

**Root cause:** the page already carries a *compound* selector for the same element, e.g. `.discuss .cover-grid { grid-template-columns: 1fr }`. That is specificity 0-2-0 against your 0-1-0, so it wins at every source position. "The later `<style>` block wins" is only true at *equal* specificity. Inline `<style>` blocks make this hard to see, because the rules sit far apart in the file and the selector that beats you does not name the wrapper class you are staring at.

**Fix:** match the specificity in your override rather than escalating to `!important`:

```css
@media (min-width: 901px) { .discuss .cover-grid, .cover-grid { grid-template-columns: 1.2fr 1fr } }
```

**Same mechanism, display toggles:** two alternative versions of one element (a wide diagram and a stacked tall one, swapped by media query) fail identically when the base rule is element+class — `svg.map{display:block}` (0-1-1) silently outbeats `.map-tall{display:none}` (0-1-0), so BOTH render stacked while every overflow gate still passes, because nothing overflows. Write the toggle rules at the base rule's weight or heavier (`svg.map-wide{display:block} svg.map-tall{display:none}`, mirrored inside the media query) and QC by element screenshot of the section at BOTH widths — a duplicated variant is a layout finding, never a numeric one. Corollary for authoring: one SVG cannot reflow, so a diagram that must stay legible on a phone needs two hand-authored layouts (same palette, same nodes), not a scaled-down wide one.

**Diagnosis — enumerate the matching rules instead of reading the source.** Grepping the HTML shows you both rules but not which one the browser picked, and guessing burns rounds. Ask the browser. Copy the page, append a probe before `</body>`, and read it back through `--dump-dom`:

```html
<script>
window.addEventListener('load', function(){
  var out = [];
  function walk(rules, media) {
    for (var i = 0; i < rules.length; i++) {
      var r = rules[i];
      if (r.cssRules && r.media) { walk(r.cssRules, r.conditionText || r.media.mediaText); continue; }
      if (r.selectorText && r.selectorText.indexOf('cover-grid') > -1) {
        out.push('[' + (media || 'screen') + '] ' + r.selectorText + ' { ' + r.style.cssText + ' }');
      }
    }
  }
  for (var s = 0; s < document.styleSheets.length; s++) {
    try { walk(document.styleSheets[s].cssRules, null); } catch (e) { out.push('SHEET ' + s + ' blocked'); }
  }
  var d = document.createElement('div'); d.id = 'PROBE'; d.textContent = 'PROBE>>>' + out.join(' | ') + '<<<';
  document.body.appendChild(d);
});
</script>
```

```bash
chrome --headless --disable-gpu --virtual-time-budget=4000 --window-size=1120,1150 \
  --dump-dom file:///C:/path/to/probe-copy.html | grep -o '<div id="PROBE">[^<]*'
```

Every rule touching that selector comes back with its media context and declarations. The one you did not know about is the answer. On a `file://` page `document.styleSheets` is same-origin, so nothing is blocked. **Probe a copy, never the file you are about to ship.** Join the output with `' | '` rather than newlines so the single-line `grep -o '<div id="PROBE">[^<]*'` captures all of it, and grep on the `<div` rather than on the marker text: the script source itself is in the dumped DOM and matches the marker first, which looks like the probe never ran.

**The same probe reads geometry, which is how you confirm the fix landed.** Add `getComputedStyle(el).gridTemplateColumns` and `el.getBoundingClientRect()` to the output. `grid-template-columns=495.266px 412.734px` proves two columns; `952px` proves one. A screenshot tells you it *looks* wrong; the probe tells you *which* value and *which* rule produced it.

### Pattern 14: Layout lost to shorthand declarations and broad selectors

- **A `padding` shorthand on one class wipes the padding another class set on the same element.** Layout containers and component classes routinely share an element (`<div class="wrap navbar">`); `.wrap { padding:0 24px }` plus a later `.navbar { padding:14px 0 }` zeroes the side padding at EVERY width, and the symptom gets reported as "the header needs padding on small screens" because that is where it is most visible. Fix with axis-specific longhands (`padding-top`/`padding-bottom`) and assert the computed style carries both axes (`getComputedStyle(el).padding` -> `16px 24px`), never just a screenshot.
- **A broad element selector inside a media query also hits the hero.** `@media (max-width:560px){ section { padding:64px 0 } }` stacks with the hero's inner wrapper padding and leaves a ~100px dead band above the h1 once a kicker is removed. Zero the specific section (`section.hero { padding:0 }`) and let the inner wrapper own the vertical rhythm.
- **Removing an element that JS binds to needs a null stub.** `var trigger = document.getElementById('x')` followed later by `trigger.focus()` in a close handler throws once the element is gone. Either delete the binding with the element, or stub it: `if (!trigger) trigger = { focus: function(){}, addEventListener: function(){} };`. Re-check console errors after ANY element removal; a clean removal still logs nothing.
- **Removing a wordmark or brand from a flex header leaves the remaining control stranded** (a burger alone on the left). Set `justify-content:flex-end` on the header row in the mobile media query so the surviving element sits where the user expects.

**Finding a headless Chrome on Windows:** if `chrome.exe` is not at the default install path, Playwright ships one at `%LOCALAPPDATA%\ms-playwright\chromium-<build>\chrome-win64\chrome.exe`. Take the highest build number present; it serves both screenshots (`--screenshot=`, with `--force-device-scale-factor=2` for legible small text) and `--dump-dom` probes. Drive it from a script file rather than a long inline shell one-liner.

**When the browser harness cannot reach a browser of its own:** the harness may report `browser-harness: daemon default didn't come up` and its log shows `fatal: DevToolsActivePort not found in [...]`, meaning it looked for a browser with remote debugging already enabled and found none. Supply one yourself, then point the harness at it with `BU_CDP_WS`:

```bash
# 1. start a headless Chrome with the debugging port open (background it)
"$CHROME" --remote-debugging-port=9222 \
  --user-data-dir="C:/.../Temp/cdp-profile" \
  --no-first-run --no-default-browser-check --headless=new about:blank

# 2. confirm it is listening and read the websocket URL
curl -s http://127.0.0.1:9222/json/version
# -> {"webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."}

# 3. export it for the session, then the browser tools attach to it
export BU_CDP_WS="ws://127.0.0.1:9222/devtools/browser/..."
```

Use a dedicated throwaway `--user-data-dir` rather than the real profile: the real profile may be open in a normal browser already, and a second instance against it fails or attaches to nothing. This recovers the whole browser toolset for the session; it is worth doing early rather than after several failed calls.

### Pattern 1: Orphaned JS After `</script>`

**Symptom:** Functions exist in the HTML source (visible via View Source or curl) but are `undefined` in the browser console. No error. Nothing happens when clicked.

**Root cause:** The HTML file has something like:

```html
<script>
// First script block with app data and rendering functions
</script>



function showHelp(){...}
function exportData(){...}
function importData(){...}
function toggleDarkMode(enabled){...}
[Podcast recorder code]
[AI Assistant code]
```

Everything after the first `</script>` is **display text**, not executable JS.

**Fix:**

```diff
-</script>
+</script><script>
+
 function showHelp(){...}
 function exportData(){...}
...
+</script>
```

Wrap the orphaned code in a new `<script>` tag. The browser will then parse and execute it.

### Pattern 2: `</script>` Inside JS String Literal

**Symptom:** A script tag appears to terminate early. Functions defined after a certain point in a script block are undefined.

**Root cause:** A string literal contains `</script>` which browsers interpret as the closing tag:

```javascript
// This string containing </script> prematurely ends the script tag!
var html = '<div class="card">...</div>';
```

**Fix:** Break the string to prevent HTML parser confusion:

```javascript
var html = '<div class="card">...<\/script>...<\/div>';
// or
var html = '<div class="card">...<' + '/script>...</div>';
```

### Pattern 3: Missing Second Script Tag in Template

**Symptom:** Generated HTML files (from templating engines, build tools, or manual editing) split JS across multiple script blocks, but one is missing an opening tag.

**Root cause:** When authoring inline JS, adding a new block of code requires `</script><script>` — not just `</script>` followed by code.

### Pattern 4: HTML Elements Inside `<script>` Tags

**Symptom:** EVERY function in a script block is `undefined` — including perfectly valid functions like `exportData`, `showHelp`, `toggleDarkMode` earlier in the same block. Clicking their buttons does nothing. Console shows one cryptic exception or nothing useful.

**Root cause:** HTML overlay markup was pasted inside a `<script>` block instead of in `<body>`:

```html
<script>
function showHelp(){...}
function exportData(){...}

<div id="podcastOverlay">   <!-- FATAL: JS parser dies here -->
  ...
</div>
<div id="helpOverlay">...</div>
</script>
```

The JS parser hits `<div` and the entire block fails — one bad line kills every function in the block, even syntactically fine ones earlier in the file. Bonus damage: the overlay divs never enter the DOM, so any code doing `getElementById('helpOverlay')` also fails at runtime.

**Fix:** Close the script tag before the HTML, reopen after if more JS follows:

```diff
 function exportData(){...}
-
-<div id="podcastOverlay">
+</script>
+
+<div id="podcastOverlay">
   ...
 </div>
```

**Detection trap:** Brace/paren balance checks pass on the broken file (the HTML is balanced too). Only actually parsing the JS (`node --check`, see Pre-Upload Validation Gate) catches it. Also check that fixing the boundary doesn't leave a stray duplicate `</script>` behind — re-count tags after the fix.

### Pattern 5: Multi-Line Single-Quoted Strings (Whole-Block Silent Death)

**Symptom:** Same as Pattern 4 — every function in the block is `undefined`. `node --check` reports `SyntaxError: Invalid or unexpected token` at the line where the string's literal newline lands.

**Root cause:** JavaScript single- and double-quoted strings cannot contain literal newlines. Common in AI-chat response objects and long content strings written by hand or generated:

```javascript
var aiResponses = {
  'skill categories': 'Essential 3 covers **five** categories:
1. **Feed** — Cook real food from scratch',
};  // SyntaxError at the second line of the string
```

**Fix:** Replace literal newlines with `\n` escapes (or convert to template literals with backticks):

```javascript
'skill categories': 'Essential 3 covers **five** categories:\n1. **Feed** — Cook real food from scratch',
```

**Companion variant:** literal newlines inside regex patterns — `.replace(/\n/g, '<br>')` written with an actual newline inside the `/.../` instead of the `\n` escape. Same whole-block death. Heuristic: any line ending in `replace(/` is suspect; also check the following line for a dangling `/g, ...)` fragment left behind after fixing.

**Escaping trap when patching:** when fixing via find-and-replace, it's easy to double-escape (`\\n` in the file = literal backslash-n in the string, which is WRONG for regexes that must match newlines and changes display output). After patching, re-extract the script and run `node --check` again, and grep the exact line to confirm single `\n`.

### Pattern 7 (renumbered from 8): SPA Screen Replacement Without Re-Activation

**Symptom:** After a user action (e.g., tapping a check-in toggle), the screen changes to completely unrelated content — like the pod creation form appearing when you were on the Week tab. No error. No crash. The wrong screen just appears.

**Root cause:** The action handler calls a `render*()` function directly (e.g., `renderWeek()`) instead of going through the app's `showScreen()` router. The render function replaces the screen's inner HTML, but does NOT re-add the CSS `.active` class to the new screen div. Without `.active`, the screen stays `display: none`. Meanwhile, another screen element that still has `.active` becomes visible — or an orphaned piece of raw HTML that was injected directly into `#view` (not inside a `.screen` div) shows through.

In a typical single-page app architecture:
```javascript
function toggleCheck(key){
  save();
  renderWeek();  // BUG: injects new HTML, doesn't activate the screen
}
```

`renderWeek()` calls `injectScreen()` which removes old `#screen_week` and appends a new one — but the new one lacks the `active` class. The CSS `.screen { display: none }` hides it.

**Fix:** Replace `renderWeek()` with `showScreen("week")`. This re-renders AND re-activates:
```javascript
function toggleCheck(key){
  save();
  showScreen("week");  // FIX: renders + activates + updates tabs + scrolls to top
}
```

**Detection:** When a UI interaction causes a screen switch to unrelated content, check the handler for direct `render*()` calls. Also check whether the app has screens that inject HTML directly into `#view` (not wrapped in `.screen` divs) — those raw elements will show through whenever any `.screen` is hidden. The fix path is always: use `showScreen()` instead of bare `render*()`. If `showScreen` does heavy work (model init, evidence rendering), add guards so it doesn't re-init every toggle — e.g., `if(!aiModelLoaded && !aiModelLoading) setTimeout(aiInitModel, 300);` inside `renderAI()`.

**Symptom:** A "Share" button using `navigator.share()` does nothing, or shows "Share cancelled" / "Share cancelled or not supported". Works in some browsers but not others, or works on desktop Chrome but fails in Android WebView / iOS Safari / PWA.

**Root cause:** `navigator.share()` is unreliable in three specific ways:
1. **`files: [file]` array** — requires Chrome on Android with specific intent handling. Fails in WebViews, PWAs, iOS Safari, and many desktop browsers.
2. **`url: dataUrl`** — most apps cannot open `data:image/jpeg;base64,...` URLs. The share target receives a broken link.
3. **WebView restrictions** — Android WebView (Capacitor/Cordova) often blocks or limits the Web Share API entirely unless specifically configured with a native bridge.

**Fix:** Always use **download-first + manual share instructions**. This works 100% of the time in every browser and WebView:

```javascript
function shareEvidence(dataUrl, type){
  var ext = type==="video"?"mp4":"jpg";
  var filename = "evidence-" + new Date().toISOString().slice(0,10) + "." + ext;
  var link = document.createElement("a");
  link.href = dataUrl;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  setTimeout(function(){
    alert("Downloaded! Now share it:\n\n• WhatsApp: Open chat → attach → Gallery\n• Telegram: Open chat → attach → Gallery\n• Instagram: Post to story or feed\n\nThe file is in your Downloads folder.");
  }, 300);
}
```

**Key rules:**
- Never rely on `navigator.share` as the only share path in a PWA or WebView
- Data URLs (`data:image/...;base64,...`) cannot be shared via `url:` — they must be downloaded first
- If `navigator.share` is used at all, always provide the download fallback as the primary path
- Include per-app instructions (WhatsApp, Telegram, Instagram, Signal) in the alert — users don't know how to attach files from Downloads

**Detection:** User reports "Share cancelled" or "Share doesn't work" on mobile. Check `navigator.share` exists (`typeof navigator.share === 'function'`) but verify the actual share payload type — `files:` and `url: dataUrl` are the failure modes.

**Verification:** After fixing, test on a real Android device or WebView. The share button should trigger a download, and the alert should show clear instructions. Do NOT test via automated browser (download blocks headless automation).

### Pattern 6: Feature Is a Degraded Stub — Prior Richer Version Exists

**Symptom:** A feature "works" but is clearly a shadow of what was intended (e.g. an "AI assistant" that is just a keyword-matching dictionary when the product called for an in-browser LLM). No errors — it responds, just shallowly.

**Root cause:** Iterative rewrites of a single-file app lost the original implementation. The deployed file is generations away from the best version, and each rewrite kept the stub because it "worked."

**Fix:** Before writing a replacement, search for the original: `session_search` for feature keywords, `search_files` for sibling artifacts (e.g. `local-ai.html` next to `index.html`). Recovering and re-integrating the proven implementation is faster and truer to intent than inventing a new one. The user saying "it is supposed to do X, see the previous setup" is the tell. After recovery, prefer linking or iframe-ing standalone artifacts over inlining them, so future rewrites can't silently drop the feature again.

**Common Pattern 6 variant — keyword-stub AI → WebLLM local model:** when the degraded feature is an "AI assistant" that's just a dictionary of canned responses, the richer prior version is usually a WebLLM (or similar in-browser LLM) integration that got reverted. Don't patch the keyword dictionary — replace it wholesale with the local-model path. See `references/webllm-single-file-html.md` for the full skeleton: importmap + dynamic `import()` + lazy load on screen entry + RAG-injected system prompt. Preserve the existing UI helpers (`addMsg`, typing indicator, chip handlers) and swap only the backend function.

### Pattern 9 (recovered): Override-then-duplicate Function Declarations

**Symptom:** After applying a structural fix to one function, the page loads but `node --check` reports `Unexpected end of input` deep in the second script. Brace/paren balance is off.

**Root cause:** When using patch tools with `old_string` / `new_string`, it's easy to write a "fix" that **inserts before** the existing function rather than **replacing** it. Result: a duplicate `function foo()` declaration followed by stale logic. Brace and paren balance holds for both copies, but the file is broken (duplicate decl → `SyntaxError: Identifier has already been declared` in strict mode, or whole-block parser failure in sloppy mode).

**Detection:** Search for any function name with `grep -c "function <name>" file.html`. If the count is > 1, you have a duplicate. For functions defined as `window.X = function`, also count `window.X`.

**Fix:** Delete the duplicate block (the older / staler copy). Verify with `grep -c` returning exactly 1 before re-validating with `node --check`.

**Recovery order:** If you patch in a loop (apply fix, validate, repeat), every patch must include `node --check` immediately afterward. Two duplicate `function aiInitModel(){` declarations both past `node --check` if no `</script>` tag is missing — the parser only sees the function keyword, not the structural issue. Brace counting alone never caught this during the session.

## Local-First UI Patterns

### Pattern: Tap-to-Contact (Member URL Fields)

For local-first pod/group apps that store member contact URLs (WhatsApp, Telegram, email, Signal), normalizing the URL into a clickable link requires three guards or you get broken hrefs:

```javascript
// WRONG — bare username becomes a relative path against the app's URL
<a href="t.me/sam">💬 t.me/sam</a>
// resolves to https://app.example.com/t.me/sam which 404s

// CORRECT — auto-prefix https:// unless already absolute or mailto:
var u = member.url || '';
var href = u;
if (u && !u.match(/^https?:\/\//i) && !u.match(/^mailto:/i)) {
  href = 'https://' + u;
}
```

Three rules:

| Input | Output `href` |
|-------|---------------|
| `t.me/sam` | `https://t.me/sam` |
| `wa.me/61400123456` | `https://wa.me/61400123456` |
| `sam@example.com` | `mailto:sam@example.com` (preserved) |
| `https://already.example` | unchanged (preserved) |
| empty | no link rendered |

Always pair with `target="_blank" rel="noopener"` and `event.stopPropagation()` if the link lives inside a clickable card — otherwise the browser bubbles the tap into the card's own click handler and the link never fires.

### Pattern: Stack-by-default Vertical Form Rows

Side-by-side form fields (name field, URL field, × button, all in one flex row) look reasonable on desktop, but overflow the screen on a real phone and often clip the rightmost field. **Vertical stacking is the default**, with side-by-side as an opt-in only above 600px width via media query.

```html
<!-- Mobile-first: stacked -->
<div class="member-field" style="margin:8px 0">
  <input class="member-name" placeholder="Name" style="width:100%;...">
  <input class="member-url" placeholder="Email or chat link (optional)" style="width:100%;...">
  <button class="remove">Remove</button>
</div>
```

This matches the member-card display logic — both display and edit use the same vertical stack. Consistency between form and rendered card reduces user friction.

## Pre-Upload Validation Gate (run before every deploy)

Before uploading a fixed HTML file, run `scripts/validate-inline-js.py` against it. One command performs four deterministic checks:

1. Extracts every `<script>` block and runs `node --check` on each (falls back to brace/paren balance if node is absent) — catches Patterns 4 and 5 with exact line numbers
2. Scans for HTML elements (`<div`, `<span`, `<button>`, `<table>`) inside script blocks — catches Pattern 4 (note: `<div` inside template literals and string-concatenated HTML in app code is NORMAL — the check flags suspicious lines for review, it's not a hard fail)
3. Audits every `onclick="fnName("` handler against defined functions (`function fnName`, `window.fnName = function`, `var fnName = function`) — catches functions referenced but never defined (e.g. a `removeServiceEvidence` called from generated HTML that was never written). `window.X = function` definitions DO count
4. Flags lines ending in `replace(/` — the literal-newline-in-regex heuristic

```bash
python scripts/validate-inline-js.py app.html
# exit 0 = clean, exit 1 = problems listed with line numbers
```

This gate replaces the slow browser-bisection loop (`typeof fn` → undefined → `eval()` chunks of script text to find the failure point — which also trips on CSP). Use the browser only to confirm the fix after the file passes the gate.

## Verification

After fixing, confirm in order — cheap checks first, browser last:

```bash
# 1. Pre-upload gate (catches 90% of failures without a browser)
python scripts/validate-inline-js.py app.html

# 2. Script tag counts match
grep -c '<script' app.html && grep -c '</script>' app.html
```

```javascript
// 3. After upload: functions are now defined
typeof exportData          // → "function"
typeof showHelp            // → "function"
// Check the FULL battery, not just the one that was reported broken:
// one broken script block often hides siblings (export, help, dark mode,
// import, podcast recorder, AI chat can all die together)

// 4. No JS errors in console (browser_console: js_errors should be [])

// 5. Feature works end-to-end — click the button, don't just check typeof
```

**Headless-browser verification trap:** do NOT validate download/alert functions by invoking them in an automated browser session. `exportData()` ends with `alert("Export complete...")` and triggers an `<a download>` click — both block headless automation, and the console call **times out after 30s** (repeated calls then fail identically, looking like a loop). The function is fine; the harness can't complete it. Verify these by (a) `typeof exportData === 'function'`, (b) reading the function body in source for the blob/download/alert sequence, and (c) asking the user to tap it on a real device. Same applies to `confirm()` dialogs (`resetApp`) and `navigator.share` — presence + source inspection is the verification, not execution.

**Duplicate function detection trap:** in this audit, after a series of patches, the file contained **two** `function aiInitModel(){` declarations in the WebLLM script block. The browser parser quietly failed in headless tests (showing offline status even after the model was loaded). `node --check` caught it as `Unexpected end of input`. Lesson: after patch loops that touch function definitions, run `grep -c "function foo"` and expect exactly 1. Patches that "insert before" rather than "replace" are the silent killer here.

**Scroll-reveal blank-screenshot trap:** pages built with scroll-reveal animations start their content at `opacity:0` (or `translateY`) and only become visible when an `IntersectionObserver` fires. A screenshot taken immediately after `browser_navigate` — or a full-page capture — can show a **blank / empty panel** even though the DOM is fully intact and the page is fine. Don't conclude the page is broken from one blank shot. Verify via the console instead:

```javascript
// Is the content actually in the DOM, just not yet revealed?
document.querySelectorAll('.reveal').length                      // total reveal nodes
document.querySelectorAll('.star, .moon, .clock').length         // key scene elements present?
document.querySelector('.clock')?.textContent                    // expected text rendered?
'IntersectionObserver' in window                                 // reveal mechanism available
```

If the DOM has everything and `IntersectionObserver` exists, the page is working — the blank was just the pre-reveal state. To *see* it, scroll (which fires the observer) or force-reveal before shooting: `document.querySelectorAll('.reveal').forEach(e=>e.classList.add('in'))`. Same trap applies to any content gated behind `requestAnimationFrame` init, canvas draws, or font-load — check the DOM/console before blaming the render.

**A text check does not prove a reveal fired.** `innerText` and `textContent` return the string at `opacity:0` exactly as happily as at `opacity:1`, so an assertion that the copy is present passes on an element no reader can see. Assert on the mechanism, not the string: count `document.querySelectorAll('.reveal.in').length` and read `getComputedStyle(el).opacity` on the element itself. The count moving from `0` before any scroll to the full total after one is the proof the observer works and the blank capture was the harness. `scrollIntoViewIfNeeded()` can land an element without ever crossing the observer threshold — scroll the document in steps (`for (y = 0; y < height; y += 400)`) and wait between them, which is also how a real reader arrives.

**Motion that silently does not run: suspect the init, not the animation.** Libraries loaded (`typeof gsap === 'object'`), zero console errors, page fully rendered, nothing animates — the classic cause is an inline `window.addEventListener('DOMContentLoaded', init)` paired with `defer` CDN tags, where the handler never sees the libraries. Replace the trigger with a self-polling init (`setTimeout(init, 50)` until `window.gsap` exists, guarded idempotent) and verify by sampling `getComputedStyle(el).transform`/`opacity` over ~1.4s rather than screenshotting a settled frame. Also: a first-pass 26px fade-up is invisible to users — masked SplitText h1 reveal plus 40px lifts is the minimum that reads as "alive". Full recipe: `references/gsap-motion-layer.md`.

**Long page, one section to inspect: isolate the section, do not stretch the viewport.** Setting the window height to the whole document (`--window-size=1120,8300`) to "capture everything" silently changes the layout: any element sized in `vh` inflates to the new viewport height, so a `100vh` hero becomes 8300px tall and the section you wanted sits nowhere near the y the DOM probe reported, if it renders at all. Force-revealing `.reveal` does not help, and a **byte-identical re-render after adding reveal CSS is the tell** that reveal was never the problem — the geometry moved.

What works: cut the section into a standalone file that keeps the original `<head>`, so it renders at a normal viewport with the real styles.

```python
head = t[:t.find("</head>") + 7]
start = t.rfind("<section", 0, t.find('id="target-section"'))
# then walk <section>/</section> depth to find the matching close
open("section_only.html", "w").write(head + "<body style='margin:0'>" + sec + "</body></html>")
```

```bash
chrome --headless --disable-gpu --hide-scrollbars --force-device-scale-factor=1 \
  --window-size=1120,900 --screenshot=section.png file:///C:/abs/section_only.html
```

When the reveal is genuine and the DOM is already correct, render the full page at a sane viewport and **crop the PNG** to the region of interest (`PIL.Image.crop((0, y1, w, y2))`) rather than re-shooting at a different size — a crop preserves the layout the real viewport produced, and crops are what you feed to `vision_analyze` for small elements.

**Tag-count false alarm: `<p` also matches `<path`.** A balance check comparing `t.count("<p")` with `t.count("</p>")` reports a large mismatch on any page with inline SVG, because every `<path` in a logo or icon counts as a paragraph opener. The imbalance is an artefact, not a broken document. Count the real tag — `len(re.findall(r"<p[ >]", t))` against `len(re.findall(r"</p>", t))` — and confirm the `<path` count separately accounts for the difference. Same class of error applies to `<a` versus `<article`/`<aside`. Do not restructure a file to "fix" a count that was never wrong.

### Pattern 13: Unbalanced `<style>` Tags Make CSS Blocks Non-Separable

**Symptom:** you remove a component's CSS from an existing page — or lift out the whole `<style>` block it lives in — and the page then renders with **no styles at all**: no heading, body `scrollHeight` of 0, the default background. The bytes look fine and every string check passes.

**Root cause:** count the tags before touching anything.

```bash
grep -c '<style>' page.html; grep -c '</style>' page.html
```

When openers exceed closers (a page missing one `</style>` is common where blocks were appended over time), a `<style>` sits *inside* an already-open style element. The HTML parser does not nest style elements — it treats everything up to the next `</style>` as **one** stylesheet. The block you are looking at is therefore not a separate block: its rules are members of the preceding block's CSS, and the page's real base stylesheet shares a parse region with them. Removing them, by any method, takes part of the real stylesheet and blanks the page.

**Rules:**

- **Never remove CSS from a page whose `<style>` tags are unbalanced.** Either add the missing `</style>` first and re-render to confirm, or leave the rules and remove only the markup and the wiring script. Dead CSS is invisible; a blank page is not.
- Removing markup and its script is safe on any page — neither sits inside a style element, and that is usually the whole of what the user asked for when they say "remove the X".
- **A balanced tag *count* is not proof of a safe edit.** Taking one whole `<style>…</style>` block out of an unbalanced page keeps both counts dropping evenly and still blanks the page, because which region gets swallowed has moved. Two structurally-clean attempts can both fail this way.

**Gate the edit by rendering it, never by diffing it.** String checks, brace balance and tag counts all pass on a page that renders blank. Render the edited file locally and assert structural invariants before committing:

```python
pg.goto("file:///" + path)
r = pg.evaluate("""() => ({
  h1: document.querySelector('h1')?.innerText.slice(0,40) || null,
  height: document.body.scrollHeight,
  sections: document.querySelectorAll('section').length,
  gone: document.querySelectorAll('.audio-player').length   // the thing you removed
})""")
assert r["h1"], "heading gone: page is blank"
assert r["height"] > 5000, "page collapsed: %s" % r["height"]
assert r["sections"] >= 3, "content missing"
assert r["gone"] == 0, "removal did not take"
```

A `scrollHeight` of 0 with a `null` heading is the signature of a destroyed stylesheet. Assert on the heading text, a height floor and a section count — asserting only that the removed thing is gone is satisfied by a blank page too.

**Recovery order once a live page is already broken:** restore the last good version from the **remote** first, before diagnosing anything. A local copy or a `git show` is not enough if your own failed attempt already overwrote the working file — a script that writes its output and *then* runs its verification gate leaves the local file broken even though the commit was refused. Fetch the file content from the remote at the last good ref, write it back as its own commit, confirm it renders, and redo the edit starting from that fetched content rather than from disk.

**Then repair the page if you will keep editing it.** Adding the missing `</style>` (and re-rendering to confirm) turns the concatenated region back into genuinely separate blocks, makes the dead CSS removable, and removes the trap for the next edit. An unbalanced page is one careless edit away from a blank screen.

**Check every page, not just the one you are in.** One loop over a site's HTML finds which pages carry the defect, so you know the blast radius before you start:

```bash
for f in *.html; do o=$(grep -c '<style>' "$f"); c=$(grep -c '</style>' "$f"); \
  [ "$o" != "$c" ] && echo "$f: $o open vs $c close"; done
```

In a template-generated site the same page is usually wrong in every copy, so expect most of the site rather than a single straggler.

**Themed app screenshots: force the colour scheme before the app boots, and suppress the dialogs.** When an app picks its theme from `prefers-color-scheme`, a headless capture only ever shows one variant, and toggling it after load fights the app's own state. Fake it *before* the module script runs by patching `window.matchMedia` in a throwaway copy of the built `index.html`. Backend-dependent apps also throw fixed-position error dialogs over the shot — hide them from an interval keyed on `position: fixed` plus a z-index threshold rather than guessing class names. Write the harness into the build output directory (usually gitignored) so it never ships, and delete it after. Recipe: `your-brand-skill/references/design-system-retrofit.md`.

## Re-Deployment Note (here.now specific)

Every publish to here.now generates a **new CDN slug**. The old URL stops resolving. After uploading a fix:

1. Upload the fixed file via the drive API
2. Publish: `POST /api/v1/publish/from-drive {driveId}`
3. Note the new `slug` from the publish response
4. Verify at the new URL: `https://{slug}.here.now/{path}`
5. If using a custom domain, update it in the here.now dashboard to point to the latest publish

## References & Support Files

- `references/debugging-orphaned-js.md` — Full worked example from a real debugging session
- `references/circles-app-audit-2026-07.md` — Case study: 8-bug audit of a single-file SPA (Patterns 1, 4, 5 all present in one file, plus missing-function and missing-CSS findings)
- `references/webllm-single-file-html.md` — WebLLM local-AI integration pattern for single-file apps: importmap, dynamic import, lazy load on screen entry, RAG injection, model picks, WebGPU pitfalls. Use when replacing a keyword-stub "AI" with real local inference.
- `references/webllm-model-selector.md` — Lite vs Pro model tier selector pattern: AI_MODELS catalog, switch logic that resets stuck loading flags, dropdown sync on render. Use when shipping WebLLM to real users who need a quality-vs-size trade-off.
- `references/web-share-api-pwa-failures.md` — Web Share API failure modes in PWAs/WebViews: `files:` array, `url: dataUrl`, WebView restrictions. Download-first pattern with per-app share instructions.
- `references/ai-chat-persistence-archive.md` — AI chat persistence pattern: localStorage-backed conversation sessions, standalone archive page for mobile, "New chat"/"History" buttons, session list with preview and deletion
- `references/local-first-pod-design.md` — Local-first pod/member management: no codes, no backend, name+URL fields, tap-to-message links with auto-https prefix, mobile-first stacked layout, and pricing removal (delete all "per seat"/$X/paid fields when going free/local-first).
- `references/copy-button-chat-responses.md` — Copy button pattern for AI chat: hover/touch-reveal button on assistant messages, strip markdown before copying, visual "Copied!" feedback, mobile-friendly.
- `references/hub-index-page-pattern.md` — Hub index page pattern for here.now (or any CDN with per-publish slugs): one permanent URL that aggregates links to all current artifacts, re-uploaded after every publish with latest slugs, fixes the "old link 404" problem for users.
- `references/backup-before-framing-change.md` — Workflow when the user says "make sure I can revert": snapshot every file you'll touch into a timestamped folder, drop a `restore.py` next to the backups that copies them all back, then make the changes. If the user dislikes the result, run the script and re-publish. **Always do this** for changes that touch AI system prompts, brand voice, or framing language — these are the most reversible-by-user changes and the highest regret cost.
- `references/three-silent-js-killers.md` — Three failing-pattern remembrances in one file: literal newlines in single-quoted strings, literal newlines in regex patterns, full-screen death on render() without showScreen(). Read once; the next time you see "everything in this block is undefined", check all three.
- `references/gsap-motion-layer.md` — GSAP motion on static pages: CDN tag shape, the DOMContentLoaded/defer init race that silently kills all animation, masked SplitText hero reveal, reduced-motion guard order, and verification-by-sampling (transform/opacity over time, never a settled screenshot).
- `references/tts-audio-injection-into-long-html-guides.md` — Pattern for embedding TTS audio players into a long single-file HTML guide (4MB+, single line). Regex with `re.DOTALL` to find chapter-opener anchors, inject player blocks after each, no preload, brand-matched inline styles. Use when adding spoken-word chapters to an existing static guide.
- `scripts/validate-inline-js.py` — Pre-upload validation gate: node --check per script block, HTML-in-script scan, onclick/definition audit, regex-newline heuristic. Run before every deploy.
