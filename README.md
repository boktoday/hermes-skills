# Hermes house skills

Six agent skills used on real static-site and agent builds, sanitised for reuse. Each is a plain `SKILL.md` in the Hermes skill format.

## Install

Give the raw URL to your agent, or install directly:

```
hermes skills install https://raw.githubusercontent.com/boktoday/hermes-skills/main/skills/<name>/SKILL.md
```

## What is here

- **responsive-static-pages** - the gate every page passes before it ships: no horizontal overflow at 360, 390, 768 and 1280, a working hamburger, real tap targets, measured with Playwright.
- **web-app-debugging** - finding script-boundary, scope and inline-JS bugs in single-file apps without a debugger.
- **verify-delivery** - checking the artefact after delivery before claiming success: live bytes, image hashes, rendered output.
- **custom-domain-wiring** - pointing a domain at a host and proving DNS actually resolved.
- **hermes-runtime-config** - making a Hermes setting take effect and persist, rather than appearing set.
- **github-repo-management** - creating and managing repositories through the GitHub API when git itself is unreliable.

## Note

These are edited copies of working notes, not a product. Read a skill before trusting it, and expect the tools they describe to move.
