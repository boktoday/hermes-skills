---
name: hermes-runtime-config
description: "Use when a Hermes setting must actually take effect."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows, macos, linux]
tags: [hermes, config, reasoning, model, verification]
metadata:
  hermes:
    tags: [hermes, config, reasoning, model, verification]
    related_skills: [hermes-agent, web-app-debugging]
triggers:
  - "Set reasoning effort / thinking level on a model"
  - "Switch the default model or provider on the Hermes gateway"
  - "hermes config set warns the key is not recognized"
  - "A config change is saved but the running session behaves the same"
  - "A cron job stalls or fails with a provider-unresponsive error"
---

# Hermes Runtime Config

For changing Hermes' own runtime settings (model, provider, base_url, reasoning effort) and
**proving the change took effect**. Load `hermes-agent` for the broad config surface; this
skill is the verification discipline plus the traps that make a saved setting silently inert.

## Rule 1: the CLI's key registry is not the runtime's

`hermes config set` validates the key against a registry and can warn
`'…' is not a recognized config key — it was saved anyway, but Hermes may not read it.
Did you mean: …`. **That warning is a false alarm whenever the runtime reads the key by a
different path than the registry knows about.** It writes the value correctly; only the hint
is wrong.

The inverse also holds: no warning is not proof the value is read. Either way, do not
conclude from the CLI. Conclude from the reader function.

## Rule 2: verify by calling the runtime's own reader

1. Locate the Hermes source tree (the directory holding `hermes_constants.py`, `cli.py`,
   `gateway/`). Its venv python is the interpreter to use.
2. Grep the source for the setting's name and find the `resolve_*` / reader function that
   consumes it — that function, not the docs, defines the real key path and precedence.
3. Call that function directly against the live `config.yaml` and print the result.

Write it to a `.py` file and run it with the venv interpreter; long inline one-liners get
refused by the command parser.

```python
import sys, os, yaml
sys.path.insert(0, r"<hermes-agent source dir>")
import hermes_constants as hc

cfg = yaml.safe_load(open(os.path.expanduser("~/AppData/Local/hermes/config.yaml"), encoding="utf-8"))
for model in ("<default-model>",):
    print(model, "->", hc.resolve_reasoning_config(cfg, model))
```

If the reader returns the new value, the setting is live regardless of what the CLI said.

## Worked example: reasoning effort

- **Key:** `agent.reasoning_effort` (global). Per-model overrides live in
  `agent.reasoning_overrides`; **per-model beats global**.
- **Reader:** `resolve_reasoning_config(config, model_name)` in `hermes_constants.py`.
- **Valid levels, ascending:** `minimal, low, medium, high, xhigh, max, ultra`. `max` is a
  real level and `ultra` sits above it — do not "correct" a request for `max` down to
  `high`. The CLI's own hint list and slash-command help are stale and omit several of
  these; trust the tuple in the source.
- **Cost:** `max`/`ultra` raise per-turn cost and latency materially. Say so when setting
  it rather than reporting the change as strictly free upside.

## Rule 3: a running session keeps its own copy

Three layers can hold this state, and editing `config.yaml` only changes the first:

| Layer | Where | Changed by |
|---|---|---|
| Global config | `config.yaml` → `agent.*` | `hermes config set`, or editing the file |
| Gateway runtime | loaded once at gateway start | restarting the gateway |
| Per-session state | `state.db`, `sessions` table, `model_config` JSON | the in-chat slash command (e.g. `/reasoning <level>`), or a new session |

So a `config.yaml` edit leaves the **current chat** on its old value. When the user asks for
a setting change, do both: write the config (durable) **and** tell them to run the slash
command for the live session, or restart the gateway. Say plainly that the session is still
on the old value rather than implying it is already in effect.

Inspect the per-session value directly — it is unambiguous about what is actually in force:

```python
import sqlite3, os, json
con = sqlite3.connect(os.path.expanduser("~/AppData/Local/hermes/state.db"))
for row in con.execute("select id, source, model, model_config, last_activity_at "
                       "from sessions order by last_activity_at desc limit 3"):
    print(row[0][-12:], row[1], row[2], row[4])
    print("   ", json.loads(row[3] or "{}"))
```

## Rule 4: a plugin install lands one session late

`hermes plugins install <name> --enable` (the flag skips the enable prompt — use it when the user
clearly wants the plugin active) resolves from the curated catalog. `hermes plugins search` lists
catalog entries, while `show` / `capabilities` / `doctor` work on **installed** plugins only — a
`not found` from `show` on a not-yet-installed name is not a failure, so don't treat it as one.

Verify an install with `hermes plugins list` (state + version), `hermes plugins capabilities <name>`
(declared tools/hooks) and `hermes plugins doctor <name>` (discovery, manifest parse, registration).

The decisive line in the install output is `deferred: tools (next session)`: the plugin is enabled,
but its tools are not callable in the current chat — they appear when a **new** session starts, and
the gateway reload the install itself performs still defers them. A skill the plugin registers can
go live in open chats immediately (the install output's `1 skills` line); tools are the deferred
part. Report both states plainly so the user knows where to reach for it.

## Rule 5: cron jobs resolve their own model — fix stalls at the cron default

Cron sessions do not run on the interactive session's model. Per-fire resolution order:
per-job `model`/`provider` > `cron.model` + `cron.model_provider` in `config.yaml` > the main
`model:` block. The scheduler re-reads config.yaml on every tick (no cache), so a config
change applies to the next fire with no gateway restart.

A main default that works interactively can still stall headless. Signature in the job's own
run log: `Provider has been unresponsive (no response received) for 5 consecutive stale
attempts`, ~16 minutes, zero API calls. Diagnose by reading the main `model:` block — if the
job hits a route never proven headless, give cron its own default instead of changing the
interactive model:

    hermes config set cron.model <model-id>
    hermes config set cron.model_provider <provider>

- Smoke-test the chosen route with a raw API call (small Python request directly to the
  provider) BEFORE re-firing the job. A chat reply proves nothing about the cron path.
- The file-edit tools refuse to touch config.yaml by design (`Agent cannot modify
  security-sensitive configuration`) — `hermes config set` is the sanctioned path; verify with
  `hermes config get cron.model` and read the raw block back.
- After a manual `run`, read the job's own output file under `cron/output/<job_id>/` — the chat
  delivery can omit parts (e.g. a trailing `MEDIA:` attachment line) that the run file keeps.

## Rule 6: identifying a "new model" — the dated name is usually a repo name, not an API id

When asked "is <model> available through our <provider> key?", never answer from the
string in the question or from press coverage. Dated checkpoint names in vendor/repo form
(`deepseek-ai/DeepSeek-V4.1-Flash-1017`) are Hugging Face repo names; providers typically
serve the checkpoint under a STABLE alias (`deepseek-flash`), so the dated string is not
callable while the model is already live on the account. Probe the provider's own list with
the key from the profile `.env` (read it, never echo it):

```python
import json, pathlib, re, urllib.request
env = pathlib.Path("<HERMES_HOME>/.env").read_text(encoding="utf-8", errors="replace")
key = re.search(r"^PROVIDER_API_KEY=(.+)$", env, re.M).group(1).strip().strip('"')
req = urllib.request.Request("https://api.provider.com/models",
                             headers={"Authorization": "Bearer " + key, "Accept": "application/json"})
print([m["id"] for m in json.loads(urllib.request.urlopen(req, timeout=30).read())["data"]])
```

Reconcile three ways: alias in the list + vendor notes saying the dated checkpoint was
pushed under that alias = **already available, nothing to change** (stable aliases advance
silently; crons on the same alias ride along). A dated id that briefly accepts calls is an
EXPIRING beta name — not a route to build on. Only a genuinely new callable id changes
config (`model.default`, `model.base_url`, `api_mode`), verified per Rules 2–3, with crons
following Rule 5. Answer with the model list as evidence and, when the alias already moved,
say "already on it" plus what changed — users asking about a fresh checkpoint need
reassurance and the deltas, not a migration plan.

## Rule 7: always-on behaviour belongs in SOUL.md

A profile's standing persona lives in `SOUL.md` at the profile root (`~/AppData/Local/hermes/SOUL.md`
for the default profile, `profiles/<name>/SOUL.md` for named ones). Its text is merged into the
system prompt of **every** session, which makes it the correct home for standing behavioural rules
the agent must carry into every task (adopted constitutions, stance, voice). Memory is wrong for
this (hard char cap, meant for facts) and a skill is wrong (loads only when its trigger fires).

- Verify it is live by checking that its exact text appears in a session's system prompt.
- Keep it lean — it costs context on every turn: the operating lines plus a pointer to the full
document, not the whole document.
- Edit with a targeted patch (append a `## section`). A full `write_file` overwrite can be refused
  by the read-tracking guard even after a fresh read — the patch path always works.

## Pitfalls

- **Back up `config.yaml` before editing it**, and prefer `hermes config set` over
  hand-editing so the file stays well-formed. A malformed config takes the gateway down.
- **A backgrounded interactive CLI silently hangs.** Commands that open a prompt (some
  `hermes … add` subcommands) return a timeout with no output when run non-interactively.
  Prefer a flag-based path; if none exists, edit the config file directly.
- **Verify the destination, not the call.** After changing anything that affects routing
  (base_url, provider), confirm a request actually resolves through the intended endpoint —
  a double-suffixed URL (`https://host/v1/v1/chat/completions`) can silently fall back to a
  different provider and still return an answer.
- **Report the level you set in the user's words.** If they asked for "thinking max", state
  which literal level that resolved to, so a mismatch surfaces immediately.
