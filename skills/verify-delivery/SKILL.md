---
name: verify-delivery
description: Verify artefacts after delivery before reporting success.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags:
    - verification
    - delivery
    - workflow
    - claims
---

# Verify before claiming delivery

A push that exits 0 has not necessarily reached the remote. A
file write that returns `verified: true` has not necessarily hit
disk in the way the agent described. A `MEDIA:` directive has not
necessarily produced an audio file at the stated path. An upload
to a remote drive may have succeeded at the API and failed at
the filesystem.

The fix is the same shape every time: **after the side effect,
read back what the system actually shows, and only report success
when the read-back matches your claim.**

## When to use

Before any response that claims an artefact was delivered — file
written, push sent, audio generated, PDF produced, upload
completed, cron created. If the message contains "pushed",
"uploaded", "delivered", "written", "generated", "live at", or any
equivalent, verify first.

If the user already saw the verification step happen inline (you
just ran `ls -la <path>` and saw the file in the same turn), don't
repeat it — that's the read-back. This skill is for the moment
*before* you claim success.

## When to apply

Before any response that contains:

- "Pushed ✓"
- "File written to <path>"
- "Uploaded to <URL>"
- "Delivered to <channel>"
- "Generated and saved"
- "Live at <url>"
- "Set to <value>" / "Configured" / "Updated the setting"

If the message contains the verb, run a verification first.

## Cost

One tool call. Trivial. The cost of *not* verifying is the user
finding out the next session that the artefact isn't actually
where you said — which they will, because they will go looking.

## Patterns by delivery channel

### Local file write

```bash
ls -la <path>            # file exists, right size?
stat <path>              # mtime confirms your write
sha256sum <path>         # round-trip the bytes if the user might re-download
```

For content-heavy files (a Markdown file you claim has a
particular structure), grep for a signature:

```bash
grep -c "<expected marker>" <path>     # ≥1 means your edit landed
```

The `write_file` tool reports `verified: true` for a write hash,
not a re-read — it confirms the bytes hit the file, not that the
content is what you described. Re-read for content verification.

**A path named in prose is not a file.** Writing "saved at
`<path>`" in a reply is a claim that the write already happened
this turn; if the artefact exists only in the chat transcript, the
claim is false and the user will go looking for a file that is not
there (a script and a Google Doc both had to be corrected this way
in one session). The order is always: create or publish with a
tool, read it back, then name the path. If you notice you have
named an unwritten path, say so plainly in the next message rather
than quietly creating it and hoping nobody checks the timestamps.

When the format prescribes line endings, **read the bytes rather than the text.** Opening a
generated file in text mode translates CRLF to LF on the way in, so a check for `"\r\n" in content`
reports the opposite of the truth and a missing-CRLF bug passes review. Assert against
`open(path, "rb").read()` for anything with a line-ending contract — iCalendar, CSV destined for a
Windows tool, or a file another program will parse.

### Git push

See `github-slow-push` — fetch, rebase if rejected, and verify
with the GitHub Contents API. Never trust a 0-exit code alone.

### here.now (or any cloud drive) upload

After the upload completes:

```bash
curl -sI <public-url>      # returns 200 and the right content-type?
curl -s   <public-url> | head -c 100    # returns the actual content?
```

The drive API may report the upload as finalised but the public
CDN may not have caught up. **A 200 is not proof the new bytes are
live — during propagation a 200 serves the previous revision.**
Retry until the expected new marker appears in the BODY, not until
the request succeeds, and cache-bust on every attempt:

```python
for _ in range(10):
    html = fetch(url + "?cb=%d" % int(time.time() * 1000))
    if "the-new-marker" in html:
        break
    time.sleep(15)
```

A loop that retries only on network failure reports a false failure
here: one deploy checked seconds after the commit showed three of
eight pages updated and read as a broken rollout, when the rest were
simply still on the old revision.

**Key the freshness check to the artefact you just produced, not to the previous
revision's signature.** A poll written against what the OLD file looked like — its old byte
length, its old pixel dimensions — satisfies itself on the very first attempt while the
edge is still serving that old revision, and reports a clean deploy it never saw. Compare
the served bytes against a hash of the file you uploaded: 8/8 matches against your own
digests is proof; a size test against the old version is not.

### A file offered for download

A curl proves the URL serves bytes; it does not prove the link on the page is wired. Click it in a
browser and catch the download:

```python
with page.expect_download() as dl:
    page.click('.card a.dl')
d = dl.value
assert os.path.getsize(d.path()) == os.path.getsize(local_file)
```

Then open what was fetched: for an archive, unzip it and assert the version inside plus one marker
the latest build introduced. That is the whole chain in one pass — the href, the host, the bytes,
the contents. A host answering `content-disposition: inline` is not a failure; an anchor carrying
`download` still forces the save.

### Audio file generation

```bash
ls -la <audio-path>       # size > 0?
ffprobe <audio-path> 2>&1 | grep Duration    # duration > 0s?
```

Piper may exit 0 with an empty WAV if the input text was empty or
the voice model failed to load. A 0-byte file is the failure mode.

### PDF generation

```python
import pypdf
r = pypdf.PdfReader("<path>")
assert len(r.pages) > 0                  # file exists, has pages
assert r.metadata.title == "expected"    # metadata landed
```

`pdf_create.py` reports `rc: 0` when it succeeds but does not check
the output is a valid PDF. A read-back with `pypdf` is the proof.

### Verifying a binary asset serves and renders

A byte check and a render check answer different questions. Both are needed, and the render check
has a false-failure mode that costs real time.

**Compare hashes, not lengths.** A corrupted file can be within a few bytes of correct, so a length
check passes something no decoder will accept. Download the served bytes and diff `sha256sum`
against the local original. (On MSYS, `sha256sum` prefixes the reported path with a backslash — strip
`\` from the output before comparing, or every asset reads as changed. Stripping is easy to get
wrong: `cut -d' ' -f1` keeps the backslash, so two identical files compare as different and a correct
deploy reads as stale. Sidestep the filename escaping entirely by hashing from stdin on both sides,
which emits a bare digest with no path at all:

```bash
live=$(sha256sum < live-file.webp | cut -d' ' -f1)
mine=$(sha256sum < local-file.webp | cut -d' ' -f1)
[ "$live" = "$mine" ] && echo MATCH || echo MISMATCH
```

When a comparison reports every item changed, suspect the comparison before the deploy — a whole-set
mismatch is an instrument fault, whereas a real stale deploy affects the items you just touched.)

**A `0` byte reading from `curl -o /dev/null -w '%{size_download}'` on a 200 is a measurement
artifact, not a missing file** — seen twice on this host for a live, correct asset. Re-check by
downloading to a real path and hashing; never report the 0 as a failure and never re-upload bytes
over an artifact reading. The same suspicion applies to any single-instrument verdict about an
asset; two instruments (size AND hash, or fetch AND decode) before you believe a bad result.

**A zero-width reading usually means "not fetched", not "broken".** An `<img loading="lazy">` below
the fold is never requested, so `naturalWidth === 0`, `complete === false` and `currentSrc === ''` —
every signal of a broken file, on a perfectly good one. Scroll it into view and wait before judging,
and read an empty `currentSrc` as a load-state fact rather than a decode failure:

```python
page.evaluate("() => document.querySelector('figure.plate').scrollIntoView({block:'center'})")
page.wait_for_timeout(2500)
page.evaluate("""() => { const i = document.querySelector('figure.plate img');
  return {nat: i.naturalWidth + 'x' + i.naturalHeight,
          lazy: i.getAttribute('loading'), cs: i.currentSrc, complete: i.complete}; }""")
```

**A `file://` render breaks root-absolute assets by construction.** Pages that reference images and
fonts as `/assets/...` resolve them against the filesystem root, so every such asset reports
`naturalWidth === 0` in a local render and the page reads as thoroughly broken. That is the render
protocol, not the page: serve the directory (`python -m http.server`) or check the live URL before
investigating any image failure seen only under a file URL.

**To separate "not fetched" from "cannot decode", decode it inside the page** — this bypasses
layout, lazy-loading and the HTTP cache at once, and is the check to trust:

```python
errors = page.evaluate("""async () => {
  const res = await fetch(path, {cache: 'reload'});
  const buf = await res.arrayBuffer();
  const bmp = await createImageBitmap(new Blob([buf], {type: 'image/webp'}));
  return {w: bmp.width, h: bmp.height, n: buf.byteLength};
}""")
```

**Use the format's own decoder for audio.** `ffprobe` reports on the header, which a truncated
stream satisfies. Add a full decode pass (`ffmpeg -v error -i f -f null -`; expect exit 0 and empty
stderr) and `volumedetect` for levels: a mean around -16 dB with a peak just below 0 dB is a real
export, while a flat floor is a silent file that passes every size check. Confirm `codec_name`,
`channels`, `sample_rate` and `duration` against the source in the same pass, so a re-encode that
trimmed content is caught rather than reported as a clean conversion.

### A file that decodes is not the file anyone asked for

Technical validity says nothing about whether an asset is the artwork intended. A file named for
one thing can hold another — a dark cover render sitting at a path named for a cream sketch plate —
and every hash, decode and dimension check passes it. Before wiring a supplied or downloaded asset
into a pipeline, read its real dimensions and sample a corner pixel or two through PIL, and compare
against what that asset is supposed to be: a cream-ground plate that samples near-black is the
wrong file, whatever its name says. A normalised thumbnail signature against the expected artwork
is the stronger version. Let the sampled numbers and colours veto the filename.

### Rendered HTML page or visual change

A grep proves the markup is in the file; nothing proves it is
visible, legible or correctly placed. Render it and look:

```bash
"$CHROME" --headless --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=2 --window-size=1120,880 \
  --screenshot=out.png "file:///C:/abs/path/page.html"
```

Then inspect with `vision_analyze`. **Judge any small element from a
`region=[x1,y1,x2,y2]` crop, never from the full-page view:** the tool
downscales a large image before you see it, so a logo mark or a hairline
border is unreadable mush in the whole-page view and perfectly fine in a
crop of the same file. A verdict of "broken" taken from the downscaled
view is a false alarm, and "fixing" it on that evidence damages working
work.

For a geometric claim — a width, an alignment, a gap — measure the captures (scan a row for
colour runs) rather than judging from the image, and on Windows treat a `--window-size=390`
capture with suspicion: headless Chrome clamps the layout to roughly 482px, so the cut-off it
shows is an artifact, not a mobile bug.

Binary discovery (system Chrome is often absent while Playwright's
bundled Chromium is present and takes the same flags), the variant
comparison harness, and the full read-back checklist:
`references/headless-chrome-render-verification.md`.

### Screenshotting a LIVE page's cards and badges

Two failure modes make a correct page look broken in a live capture, and both are the capture's
fault:

- **Lazy images read blank.** `<img loading="lazy">` below the fold is never requested, so a grid
  shows empty boxes while the page is fine. Force every image eager, then gate the shot on the
  loaded count:

  ```python
  page.evaluate("() => document.querySelectorAll('img').forEach(i => { i.loading = 'eager'; })")
  page.wait_for_timeout(5000)
  print(page.evaluate("() => Array.from(document.querySelectorAll('img')).filter(i => i.complete && i.naturalWidth > 0).length"))
  ```

  Shoot only when loaded == total; a partial count is a timing fact, not a defect.
- **A sticky header eats the top of the element you scrolled to.** `scrollIntoView({block:'start'})`
  parks the card's top edge under the fixed nav, so a corner badge sitting at `top:40px` inside it
  is occluded and reads as missing — easy to misdiagnose as a CSS bug. Offset after scrolling:

  ```python
  page.evaluate("() => { const el = document.querySelector('.target'); el.scrollIntoView({block:'start'}); window.scrollBy(0, -110); }")
  ```

And on a live deploy, confirm the change is actually SERVED (marker present in the fetched body)
before capturing — then the screenshot verifies the deploy rather than the local working copy.

### A rendered video deliverable

A render that exits 0 proves frames were drawn, not that the video is right. Caption, overlay and
asset faults never surface in build logs, and every check that reads only sizes and durations
passes a wrong picture, so the frames are the delivery gate. Extract stills at the beat boundaries
and both ends, paste them into one contact sheet, and look at it:

```bash
for t in 1.5 9.5 27 34 48.5; do ffmpeg -v error -ss $t -i out.mp4 -frames:v 1 -y "frame-$t.png"; done
```

What the sheet settles that nothing else does:

- **Frames from different timestamps that share one exact byte size are almost certainly
  identical.** Whatever the timeline says, the picture is not changing there — the tell for a
  caption or overlay layer rendering one static block on every frame.
- **A frame much larger than its neighbours marks the beat that carried real imagery**, exposing
  where picture content landed against where it was intended.
- Text is judged here and nowhere else: unreadable sizes, lines overflowing the canvas and
  off-centre placement are invisible to pipeline success. Read the sheet as an image; no log
  substitutes for it.
- Confirm the sound with the same full-decode plus `volumedetect` pass as the audio section above;
  a video whose narration was dropped at the mux stage passes every size and duration check.

Budget two or three rebuilds for caption and placement polish, and batch every visible fault into
one pass — renders of a minute-scale piece cost minutes each, so iterating one property at a time
multiplies the wait by the number of faults.

### A layout bug the user can see and you cannot reproduce

When the user sends a screenshot of a page looking wrong and your own captures keep coming back
blank or partial, **stop capturing images and measure the DOM instead.** Two reads settle most of
it and neither needs a good screenshot:

```python
info = page.evaluate("""() => {
  const el = document.querySelector('.the-element');
  const parent = el.closest('.the-wrapper');          // the container it SHOULD be in
  const r = el.getBoundingClientRect();
  return { left: r.left, right: r.right, width: r.width,
           parentFound: !!parent,
           parentLeft: parent ? parent.getBoundingClientRect().left : null,
           parentRight: parent ? parent.getBoundingClientRect().right : null };
}""")
```

- **An ancestor that does not exist is the finding, not a detail.** An element that measures a
  plausible width but has no expected parent is not inside the thing supplying its background,
  padding and centring — which is exactly what a screenshot showing "the block sits on the page
  background instead of the section background" looks like. `closest()` returning `null` is what
  two rounds of squinting at captures failed to reveal.
- **Compare the element's edges against its PARENT's edges at each breakpoint**, and require
  equality when the design says they should match. Print the numbers rather than describing them.
- **A negative horizontal margin only stays on the page if the element is inside the padded parent
  it is cancelling.** An element carrying `margin:0 -110px` placed BETWEEN two sibling wrappers
  cancels nothing, is wider than either wrapper, and overhangs the page background on both sides.

Related authoring rule this verifies: when injecting a block into a page built from sibling wrapper
elements, insert it INSIDE the wrapper, immediately before that wrapper's closing tag — never in
the gap between one wrapper's close and the next one's open. The two placements look identical in
a diff and to a `grep` for the anchor string, which is why the mistake survives review.

**When the user says it is still wrong, believe the report over your evidence.** Two rounds of
"my screenshots look fine" against a user saying "no good" means the instrument is broken, not
their eyes. Switch instruments immediately — measure geometry instead of capturing pixels — rather
than re-running the same capture at another window size.

When the change is a layout one, apply the same discipline to the page's own CSS context: a
full-bleed element achieved with a negative margin leaves its caption and any sibling text bare
unless they are given matching padding back at every breakpoint. Check the caption's left edge
against the body copy's left edge, not against the block above it.

Then check its COLOUR against the surface it actually lands on. A block styled once on a shared
class is dropped into whichever container variant the page happens to use, and a colour picked
against one variant fails on the other: a caption set at 5.9:1 on a dark panel measured 1.5:1 on
the light panel the very same rule also styled, which is unreadable. Scope the colour to the
container (`:not(.the-dark-variant)`) rather than setting it once on the shared class, and compute
the ratio instead of judging it by eye:

```python
def contrast(a, b):
    def lum(c):
        c = c / 255
        return c / 12.92 if c <= 0.03928 else ((c + 0.055) / 1.055) ** 2.4
    la = 0.2126*lum(a[0]) + 0.7152*lum(a[1]) + 0.0722*lum(a[2])
    lb = 0.2126*lum(b[0]) + 0.7152*lum(b[1]) + 0.0722*lum(b[2])
    hi, lo = max(la, lb), min(la, lb)
    return (hi + 0.05) / (lo + 0.05)
```

Measure the SPECIFIED colour, not the rendered pixels: anti-aliasing blends text toward its
background, so sampling a screenshot reports a worse ratio than the CSS actually achieves and sends
you chasing a problem that is not there.

### Identifying which artefact the user actually approved

When you send several variants (candidate images, two drafts, alternative cuts) and the user replies
with one of them, **do not trust the reply's position or ordering to tell you which.** Chat clients
renumber, reorder and strip the filenames you sent, so "this one" arrives with no reliable handle on
it. Compare the returned artefact against your candidates and pick the match by measurement:

```python
from PIL import Image, ImageOps
import numpy as np

def sig(path, n=24):                      # normalised greyscale thumbnail signature
    im = ImageOps.fit(Image.open(path).convert("L"), (n, n), Image.LANCZOS)
    a = np.asarray(im, dtype=np.float32)
    return (a - a.mean()) / (a.std() + 1e-6)

d = {name: float(np.mean(np.abs(sig(cand) - sig(returned)))) for name, cand in candidates.items()}
```

The candidates are visually distinct, so the distances are categorical rather than marginal — a real
match lands an order of magnitude below the others, and the gap is the proof. Act only on the
identified one, and name which you identified in the report so a misread is visible immediately.

Getting this wrong is expensive in a way a normal verification miss is not: the wrong variant gets
published, or a whole set is generated in a style the user had just rejected.

### Claiming a page is installable (PWA)

Do not infer installability from the presence of a manifest. Ask the
browser that decides:

```python
cdp = await page.context.new_cdp_session(page)
await cdp.send("Page.enable")
res = await cdp.send("Page.getInstallabilityErrors")
print(res["installabilityErrors"])
```

A page can link a manifest carrying a name, 192px and 512px icons,
`display:standalone` and a `theme_color`, and still be one field
short. The error list names the missing one (`start-url-not-valid`
for an absent `start_url`); a page with no manifest at all reports
`no-manifest`. Filter `in-incognito` out of the result — it is an
artefact of running headless or automated, not a defect on the site,
and taking it at face value sends you hunting a problem the site
does not have.

The same distinction applies when explaining the feature: browsers
will install ANY page by hand from the menu (`Install Page as App`
on desktop, `Add to home screen` > `Install app` on mobile), while
the automatic prompt requires the criteria. Establish which of the
two the user actually means before answering.

### Telegram / WhatsApp send

The chat adapter reports success or a timeout. A timeout means
the file was generated but the send failed — the user did not
get it. To verify:

```bash
# Check the file the send claimed to attach
ls -la <attached-file-path>
# Check the archive (if your flow archives sent items)
grep -c "<message-id>" <archive-log>
```

A "delivered ✓" report without a real send confirmation is a
ghost-delivery.

**"Attachment didn't come through" is a first-class failure mode for video.** The adapter can accept a
`MEDIA:` line and the file fail to arrive at the user, with no error anywhere on this side — a 13MB
MP4 did exactly that, and a 4MB 720p re-encode of the same cut did not visibly land either, while a
sub-megabyte MP3 sent in the same style arrived. So do not answer "it sent fine" and do not resend
the identical file. The working ladder:

1. Re-encode smaller first: `ffmpeg -vf scale=1280:720 -c:v libx264 -crf 28 -preset medium -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart` and keep the result around 3–5MB.
2. Send that as the `MEDIA:` line, on its own line, with no bold or punctuation around it.
3. If it still does not arrive, offer a link (public host or object URL) instead of another resend, and say plainly that the file was delivered as a link because the attachment route failed.

Audio is the reliable channel for a spoken deliverable (an MP3 with `[[audio_as_voice]]` plays inline);
when the artefact is a short video and the attachment keeps failing, say so early rather than iterating
encodes for half an hour.

### Cron job creation

After `cronjob(action="create", ...)`:

```python
cronjob(action="list")   # does the new job appear with the right schedule?
```

Cron registries have been observed to silently drop jobs when the
schedule string has a malformed field.

### Settings or config change

A config write is **not** verified by re-reading the file you just
wrote. That proves the file holds the value, not that anything
reads it. Resolve the value through the same reader the runtime
uses:

```python
# the reader the runtime actually calls for the key at hand,
# not yaml.safe_load(config.yaml)
print(resolve_reasoning_config(CLI_CONFIG, model))
```

Three failure modes this catches, none of which raise an error:

- the key name is wrong, so the value sits in the file unused
- the CLI warns a key is "not recognized" for a key the runtime does
  consume, so a correct write looks like a failure; the same drift
  runs the other way, and a warning-free write to a key nothing
  reads is silent
- the process read the config at startup and cached it, so a session
  already running keeps the old value no matter what the file says

Report a config change as two facts, not one: what the file now
holds, and what the running process will actually use. If the live
process needs a restart or an in-session command before it picks
the value up, say that in the same breath. "Set to X" while the
running session still uses Y is the same class of false claim as a
phantom push.

### A value that appears in more than one place

Prices, dates, product names, the weekday of a session, a phone number, a brand token.
These get duplicated across surfaces that drift independently, and the user usually
names only one page when asking for a change.

Brand tokens are the ones most often recited from memory instead of read, because they feel
like stable house facts. They are not. The same variable name can carry a different value, or
be absent entirely, on different pages of one site, and a name that reads as a core token on
one page is often a literal somewhere else. Read the CSS `:root` blocks and the per-page font
`<link>`s before answering any palette or typography question, and say which page each value
came from. Two separate brand facts reported from memory in one session were both wrong. Grep the repo, then read the
LIVE surfaces, because the two disagree and the repo is the more misleading
of the pair.

```bash
# every local occurrence with context
# search_files for the literal figure across *.html *.txt
# what is actually served, per surface
curl -s "https://site/"            | grep -o '.\{0,70\}\$[0-9]*\.[0-9][0-9].\{0,70\}'
curl -s "https://site/llms.txt"
curl -s "https://separate-product-domain/"   # a different domain is a different surface
```

Traps, in the order they bite:

- **Archived variants in the repo mislead you.** `index-a.html`, `_live.html`,
  `index-archived-*.html` and friends all match the grep and none are served.
  Establish which files are live (fetch the live page and diff its href set)
  before concluding "the figure is wrong in six places".
- **The odd one out may not be the page the user named.** When three surfaces
  agree and one disagrees, state which is which rather than silently
  normalising the named page to match the majority. Restoring consistency and
  changing the value are different acts, and the user must know which you did
  and which surfaces you touched.
- **Machine-readable surfaces count as live surfaces.** `llms.txt`, JSON-LD
  `offers`, meta descriptions and `og:` tags are read by answer engines and
  assistants. A stale figure there is quoted back to users as fact and nobody
  sees it on any page. Update them in the same pass, or say you did not.
- **Fix the unit and its dependent copy, not just the number.** Turning
  `$8.88 / per month` into `$44.88 / one time` leaves "Cancel anytime." reading
  as nonsense. Grep the same block for `month`, `cancel`, `subscribe`, `renew`,
  `lock-in` and rewrite or delete them.
- **Report remaining disagreements you did not touch.** A weekday or term that
  still conflicts across surfaces is worth one line, with the counts, even
  though it is out of scope. Which side is correct is a business fact you
  cannot derive from the repo, so flag it rather than picking one.

### A structural check that fails after your edit

Before treating a failed structural assertion as your regression, run the same check against the
pre-change revision. A page can carry a defect for months that only surfaces when an edit makes
you count something for the first time, and the tell is a discrepancy of identical size before
and after your change.

```python
after = counts(open(path).read())            # your edited file
before = counts(fetch(live_url))             # the currently SERVED revision
bad_after  = {k: v for k, v in after.items()  if v[0] != v[1]}
bad_before = {k: v for k, v in before.items() if v[0] != v[1]}
assert bad_after == bad_before, "your edit changed the structure"
```

The live URL is the convenient baseline whenever the deploy is not your own: it is the last
revision the user can actually see. A matching delta proves your edit was neutral, which is a
different claim from "the page is fine", and it is the one that keeps you from either chasing a
phantom regression or shipping a silent fix.

Keep the two apart in what you ship. An incidental fix for a defect you happened to find is its
own change with its own before-and-after render check, not a passenger on a content commit:
closing an unclosed `<style>` changes how CSS parses on every page it touches, so bundling it
with a copy change leaves nobody able to say which of the two altered the rendering. Report the
defect with its evidence and offer the fix separately.

Count tags with one primitive and sanity-check the output's shape. A classifier whose branches
pad one side (`"OPEN " if … else "CLOSE"`) silently inverts every comparison against the
unpadded literal, and a depth counter that marches monotonically negative is reporting on the
script, not the document. When a check's output looks absurd, suspect the instrument before the
artefact.

### A generated artefact is a snapshot of the moment it was built

An offline bundle, an exported zip, a rendered PDF, a downloaded archive: all are point-in-time
copies. Every later edit to the source leaves them behind, and nothing warns you — the file still
exists, still serves 200, and still looks correct. A download link is therefore a claim about
content, so linking one built before the last few changes publishes a stale product.

Two duties whenever a page offers one:

- **After any change to the source content, treat the artefact as invalidated.** Either rebuild it
  within the same change, or say plainly what it lacks — which chapters, which audio, which fix.
  Saying so costs one sentence; a user who downloads a stale bundle has no way to discover it.
- **When you add a prominent link to one, re-check that it still matches.** A button makes the
  artefact load-bearing in a way a footnote does not, and the link is the part you were asked for,
  so the staleness behind it is easy to miss.

Rebuild over the same URL where the publisher allows it, so existing links keep working instead of
multiplying release URLs with each refresh.

### Replacement assets generated but never published

Replacement imagery is the easiest work to leave behind. The files land in a temp folder, the
live site keeps serving the previous artwork, and nothing looks broken — so nothing prompts a
second look, and only the user notices. A report that the old artwork is still live is an
unfinished job, not a bug in the pipeline.

- **Generating a replacement is half the task; publishing it is the other half.** Never end a
turn with new assets on disk and the live URL untouched, however good the generation looked.
- **Ship by overwriting the existing filenames.** When the replacement can carry the same
  names as the files it replaces, the swap needs no page edits at all and no HTML is at
  risk. Prefer that over renaming and rewiring every reference.
- **Regenerate the derivatives in the same pass.** Thumbnails, crops and grid previews were
  built from the old asset; leave them and the index shows one style while the pages show
  another. Match the derivative's exact existing pixel box so no layout shifts.
- **Sweep the mirrors and bundles.** Offline bundles, downloadable zips and mirror folders
  carry their own copies of the same art; refresh them in the same pass or the next rebuild
  reintroduces the old version.
- **Verify the whole set, pages and derivatives.** Eight fresh plates with eight stale
  thumbnails is a half-shipped swap; check both lists against the files you produced.

### Reporting a pilot as a pilot

When a change is deliberately built on one item of a set and the rest await sign-off, say so in
the first line rather than as a closing offer. A polished single item reads as a finished
rollout: the user sees the result, approves the format, and reasonably assumes the whole set now
follows it. State the scope explicitly — how many are done, how many remain — and treat the
rollout as wanted the moment the format is approved, so offer it in the same breath.

### Resuming a deferred request

"Can you follow up on that thing I asked about?" publishes the wrong artefact in two
ways, and both look like success from the inside.

**Recover the original ask from history, not the paraphrase.** Search the sessions,
then scroll around the matched message id: the request and the artefact you prepared
sit several messages apart, and the paraphrase drops exactly the specifics that matter
(a title, a variant, which of two files). The follow-up also confirms what the user
believes they asked for, which is not always what the log shows.

**Then check you are on the right thread at all.** Two parked requests of the same
shape (both "turn it into a Doc", both stalled on the same unanswered question) can
live in one session, and the user names the one they remember rather than the one they
mean. When the follow-up and the thread in front of you disagree, say which you
concluded and why BEFORE the write — the write is the expensive half to get wrong, and
one sentence of disambiguation is cheap. Do not extend an old approval to the other
parked request just because you noticed it; raise it instead.

### Verifying an edit that comments something out

A grep proves a string is present, not that it is live. To a regex an HTML
comment is just text, so counting `<audio id="promo-audio"` returns 1 for a
block the browser will never parse, and a `//`-commented JS call counts the
same as a working one. One session hit this twice in a row, on a
decommented-out audio player and a hidden button, and both times the first
check reported a false "still live".

Skip matches that sit inside a comment, and assert on the count:

```python
def live(doc, snippet):
    n = 0
    for m in re.finditer(re.escape(snippet), doc):
        before = doc[max(0, m.start()-6000):m.start()]
        if before.rfind("<!--") > before.rfind("-->"):
            continue                       # inside an HTML comment
        line = doc[doc.rfind("\n", 0, m.start())+1:m.start()]
        if line.strip().startswith("//"):
            continue                       # inside a JS line comment
        n += 1
    return n
```

Check the restore path in the same pass: the marker comment you deliberately
left behind should still be findable, so a later "uncomment it" really is a
one-line job, and say so in the report.

Belt and braces when it matters: load the served page in a browser and count
the elements in the parsed DOM (`doc.querySelectorAll('audio').length`). That
is the one check markup that is present but inert cannot fool.

**A check that reports "missing" deserves the same suspicion as one that reports a pass.** An
existence test greps for a needle you composed rather than copied; if that needle appears nowhere in
the artefact — a handler looked for as `webhook/` where the URL actually says `webhooks/` — the
check reports a working feature as absent and invites you to "fix" something that was never broken.
Copy the needle out of the artefact itself, and grep your own source for it once before acting on
a negative.

### Matching an output the user has already produced by hand

When the ask is "do X in this format" and the user has made one example themselves, that example IS
the acceptance test, not a vibe to imitate. Diff your generated output against it before shipping
anything.

Compare at the level that matters, and strip the decoration first. For formatted text, drop the
markup markers (pause tags, bullets, braces) and compare WORD SEQUENCES with
`difflib.SequenceMatcher` — sequences, not sets, so a reordering reads as a difference instead of
passing:

```python
def words(t):
    return [w for w in re.split(r"\s+", t.replace("<#0.5#>", " ")) if w]
difflib.SequenceMatcher(None, words(reference), words(generated)).ratio()
```

Treat anything short of `1.0000` as a live question rather than a rounding error: each remaining
difference is either a rule you have not implemented or a deliberate deviation you must name
out loud when you report.

**Where a second, independently produced output exists, check that too.** Two routes agreeing
word-for-word is the real evidence; one method reproducing itself is not.

Then assert the per-instance invariants a word diff cannot see. A "the string appears exactly
once" check reports a false problem whenever a value legitimately appears twice — count the
occurrences and account for each one instead of trusting the count.

### A change built but not yet published

Local verification is not delivery. A page edited, rendered and measured on disk is invisible to
the user until the commit exists and the live URL serves the new bytes — and a turn can end with
the work at exactly that point when a second request arrives mid-stream.

- **Publish before you switch tasks.** When a finished-but-unpushed change is in hand and a new
  request lands, push it first — it is seconds of work — then start the new one. Nothing marks
  the boundary between the two except you, and the new task reads as urgent while the unpushed
  one silently drops out of the report.
- **If it is not live, the report says so.** "Built and rendered, not yet pushed" is one
  sentence; a locally-verified change described without that qualifier reads as delivered.

When the user reports that a change is not visible, check for a missing push BEFORE re-checking
the page. Re-rendering locally re-confirms the fact that was never in question.

### Acting on a defect report produced without rendering

Feedback on a page frequently comes from a tool that read extracted SOURCE text, not the rendered
document. Source-readings produce a stable family of false alarms, and "fixing" them burns a
deploy on working things:

- **Hidden-by-CSS elements reported as visible** — status and error divs that are `display:none`
  until triggered read as on-screen junk in the source extract.
- **Links reported as missing** because the extract carried anchor TEXT but no hrefs.
- **The reviewer's own chrome attributed to the site** — section headings, list numbers and
  navigation labels that exist nowhere in the HTML ("Page Links" was one such phantom).
- **List numbers/labels silently dropped** by the extractor, then read back as inconsistency.

Procedure: verify EVERY claim against the rendered page in a real browser (computed styles,
`querySelectorAll` counts, live hrefs) BEFORE editing; keep a false-alarm list with its evidence
separate from the real changes in the report. Genuinely vestigial hidden elements (dead status
nodes from an earlier launch state) are still worth DELETING so the next source-reader stops
flagging them. Treat any suggested change to statements of fact about the user (bio lines, org
claims) as a draft for their confirmation rather than a commit.

## Anti-pattern

"Looks like it worked. Reporting success." The user has flagged
this twice on this host — once for a phantom push ("did you push
all this to the repo?"), once for content that wasn't where it was
claimed ("I thought you make these chapters full content and put
them into the repo"). Don't be the third.

If the read-back fails, the report says so. The cost of saying
"the push didn't land, here's what I see" is one extra sentence.
The cost of saying "pushed ✓" when it didn't is the user trusting
the wrong report.