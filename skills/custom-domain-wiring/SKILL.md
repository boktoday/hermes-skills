---
name: custom-domain-wiring
description: "Use when wiring, verifying or debugging a custom domain."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [dns, domains, vercel, siteground, rdap, doh, deployment]
    related_skills: [static-site-deployment, github-content-push, epilogue-life-project]
---

# Custom domain wiring & diagnosis

Attaching, verifying and debugging custom domains for deployed sites. the owner's zones are managed at SiteGround (`ns1/ns2.siteground.net`) for several domains, with sites on Vercel; most "domain problems" are one of: record not set, registrar delegation missing, client cache, or a verification step. Diagnose from the OUTSIDE first (public DNS + registry + served headers) — you cannot see his hosting dashboards, and panels can look correct while being inert. Reply with exact record edits, not vague "wait for propagation".

## When to use

- "Point this domain at the site" / a new subdomain is needed for a project.
- "Why is this domain taking so long to update?" / a domain sits "pending" in a host's panel.
- A domain serves the old site (or nothing) after records were edited.
- After any DNS change: the verification pass before telling the user it is live.

## Diagnosis order (before touching anything)

1. **Resolve from outside** — DoH A/CNAME/NS/TXT via BOTH `dns.google` and `cloudflare-dns.com` (they disagree mid-change; prefer DoH over the local OS resolver, whose cache lies).
2. **Check registry delegation FIRST when the complaint is "slow to update"** — `https://rdap.org/domain/<apex>` gives registry nameservers, status, registrar, expiry; also query NS via DoH and read status 3 (NXDOMAIN, authority `a.gtld-servers.net`) as undelegated.
   - **Empty registry nameservers = every record in the user's zone panel is invisible to the internet.** No amount of waiting fixes it; the nameservers must be set at the REGISTRAR. This is the observed root cause of a "why is it taking so long" — the zone was perfect and inert.
   - Registered is not delegated: RDAP can show an active, locked registration (`client transfer prohibited`) while the whole domain answers NXDOMAIN in DNS.
3. **Compare records to the host's expected targets** (table below).
4. **Fetch and read headers** — `server: Vercel` / `x-vercel-id` = Vercel served it; `server: cloudflare` + `cf-ray` = Cloudflare (or the old host's platform) is in the path — a domain still on the old host shows this while "pending" in Vercel.
5. Only then form the verdict: undelegated / record not set / wrong target / propagation (rare) / client browser cache.

## Platform targets (verified working)

- **Vercel**: apex A → `216.150.1.1` (current anycast; legacy `76.76.21.21` also valid); subdomain CNAME → `cname.vercel-dns.com`, or the per-project target the dashboard hands out (`<hash>.vercel-dns-0NN.com`) — both work. Add the domain in the project's settings FIRST, DNS after. With apex + www both configured, one redirects to the other (epilogue.life: apex → www).
- **SiteGround zones** (the owner's DNS for his .com.au / .life domains): NS `ns1.siteground.net` / `ns2.siteground.net`; TTLs are short — record edits apply in minutes once delegation exists.
- **Verification TXT** (`_vercel.<name>` → `vc-domain-verify=…`): only appears when the name overlaps across accounts/projects; no TXT record = no verification hurdle — do not invent one.
- **Netlify app subdomains** (`app.<domain>` → `*.netlify.app`): leave untouched when moving the marketing domain.
- **Registry NS changes apply in minutes, not days** — treat "24-48h propagation" claims as a last resort, not a default.

## Before flipping a record off an old site

Archive the outgoing site FIRST — fetch the page, download its referenced assets (`src`/`href`), zip it. Small script, runs in a minute; then flip the record with nothing at risk. (Proven on the old iam.aiorchestrator.com.au before its DNS swap.)

## Verification pass after wiring (all of it)

- DoH both providers: apex + www A/CNAME agree.
- HTTPS both names: 200, expected final URL (redirect direction), correct title, a NEW-content marker string, `x-vercel-id` present.
- Fetch the key assets and **byte-size match** them against the local new files — catches CDN leftovers and confirms which build is actually live.
- Fresh-browser (Playwright) render at 390 + 1440: overflow gate + all `naturalWidth > 0`.
- TLS valid through the chain; cert issued for both hostnames.

## Finding the live URL of a project

Before probing guessed host subdomains (`<project>.vercel.app` variants), check the GitHub repo's `homepage` field: `GET /repos/{owner}/{repo}` → `homepage`. The deploy URL is often recorded there — name-guessing burned a dozen 404 probes while the homepage field held it all along.

## Pitfalls

- **"Taking so long to update" is almost never propagation.** Check delegation and record values first; registry NS changes land in minutes.
- **Editing records while delegation points elsewhere (or nowhere) is invisible work.** Diagnose at the registry, not in the panel.
- **Two DNS views disagree mid-change** (DoH vs local cache): trust the DoH providers.
- **Do not guess project URLs by hammering name variants** — use the homepage field or ask.
- **Do not flip a record off a live old site without archiving it first.**
- **A name in a non-ICANN namespace is not a business address.** Agent-identity registrars sell names inside Handshake roots (`.agent`, `.chatbot`, `.bpo`, `.boss`) for a few dollars and take crypto or their own credits. Such a name resolves only through Handshake-aware resolvers, is not indexed like a `.com`, and will not open on a normal phone or a customer's data bundle. Before buying an unfamiliar TLD, check the root — `https://rdap.verisign.com/com/v1/domain/<name>` for .com, or a DoH NS query for others; no ICANN record means the name is an experiment, not a brand. Buy the .com for the business and treat the novelty TLD as a fiver-sized experiment.
- The user may paste a hosting-dashboard URL (e.g. `vercel.com/<team>/<project>/settings/domains`); you cannot browse authenticated. Ground everything in external evidence and reply with exact record edits.

## Supporting files

- `scripts/dns_check.py` — one-shot outside-in probe: per name, A/CNAME/NS/TXT across both DoH providers; registry RDAP for the apex (nameservers / status / registrar / expiry, with an UNDELEGATED flag); then an HTTP fingerprint of `https://<name>/` (status, redirect target, title, server / x-vercel-id / cf-ray, bytes). Run: `python dns_check.py <apex> [more names ...]`.
