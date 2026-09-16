# Nous hermes-agent full-repo survey (for Ray)

**Checkout:** `adastra-labs/hermes-agent` fork of `NousResearch/hermes-agent`  
**Tree surveyed:** `origin/main` at `03b0c79472` (2026-09-16) plus `origin/feat/cdp-endpoints-session-map`  
**Upstream PRs (via `gh`, NousResearch/hermes-agent):** #112937, #108914, #49691, issue #49693, issue #92524, follow-ups #112407 / #97859  
**Method:** directory map, grep (portal, bot_desktop, Screen, /learn, cdp_url, cdp_endpoints, CapSolver, lease, computer_use), file reads, `git diff origin/main...origin/feat/cdp-endpoints-session-map`, `gh pr view` / file lists.  
**Scope:** read + report only. No feature work.

---

## Executive answer

**Did prior work “read the repo”?** **Partial, and mostly the CDP surfaces.**  
#112937’s code and tests are a real, local read of `tools/browser_tool_cdp.py`, `browser_use_cli.py`, `/browser connect`, and `session=` cache keys. Its *claim* that this complements Bot Screen #108914 is **not** backed by reading `tools/bot_desktop/` or the Desktop Screen UI — those files **do not exist on `main`**. The PR body and `cdp-endpoints.md` cite #108914 as a neighboring product surface (and as an ADA-lab operator constraint), not as a module they patched.

**Complements #108914 — yes / no / partial?** **Partial.**

| Claim | Verdict | Why |
|---|---|---|
| Different jobs; not a Screen/Start button | **Yes** | #112937 is a config+CLI name→CDP map. #108914 is a per-profile Xfce+TigerVNC seat streamed into Desktop with a lease. |
| “Do not move CapSolver onto Bot Screen DISPLAY” | **Operator policy, not code** | Neither PR implements that fence. #108914 **explicitly unfences** user-supplied CDP. |
| Lease-fence CapSolver CDP while a human holds Screen | **No (and Bot Screen disagrees)** | `run_fenced` in #108914: “Cloud / user-supplied CDP sessions are a different browser and run unfenced.” Follow-up #112407 repeats that. #112937 never imports `bot_desktop`. |
| Does not duplicate #49691 / #49693 | **Shallow / wrong-ish** | Same *category* as #49691 (`name → CDP URL`). Different keys (`cdp_endpoints` vs `profiles`) and different fail policy vs #49693. Real overlap is here, not Bot Screen. |
| Merge-safe with Bot Screen | **Mostly, with three shared files** | Overlap: `hermes_cli/config_defaults.py`, `tools/browser_tool_session.py`, `website/sidebars.ts`. #108914 is already `CONFLICTING` vs `main`. |

**Blunt:** calling this a complement “after reading the surfaces” was accurate for *product intent* (don’t bolt a Screen button onto a stay-put Chrome) and **too thin** for *Bot Screen internals*. The interesting conflict is with **#49691 / #49693**, not with Teknium’s Xfce pane.

---

## 1. Product map (what this repo ships)

Hermes is one agent core with many fronts. Root `AGENTS.md` and `README.md` are the inventory; the filesystem is canonical.

### Surfaces (same agent, different UIs)

| Surface | Path / entry | Role |
|---|---|---|
| **CLI** | `cli.py`, `hermes_cli/`, `hermes` | REPL, slash dispatch (`_SLASH_DISPATCH`), `hermes setup`, `hermes tools`, `hermes portal` |
| **TUI** | `ui-tui/` (Ink) + `tui_gateway/` | `hermes --tui`; JSON-RPC over stdio |
| **Desktop** | `apps/desktop/` (Electron) | Native chat; talks to `tui_gateway` over WebSocket (`apps/shared`). Own renderer — **does not embed TUI or dashboard**. Spawns pooled `hermes serve`. |
| **Web dashboard** | `web/` SPA + `hermes_cli/web_server.py` + `hermes_cli/web_routers/` | Admin panel (`hermes dashboard`, `:9119`). Chat tab embeds the **TUI over a PTY**, not Desktop. |
| **Messaging gateway** | `gateway/` + `gateway/platforms/` | Telegram, Discord, Slack, Signal, WhatsApp, QQ, Weixin, webhook, API server, … (~20 adapters). Survives Desktop; detached process. |
| **ACP** | `acp_adapter/` | VS Code / Zed / JetBrains |
| **Cron / kanban** | `cron/`, `hermes_cli/kanban*.py`, `tools/kanban_tools.py` | Scheduled jobs; Bot Mode routines are namespaced cron |

### Agent waist + edges

| Area | Path |
|---|---|
| Turn loop | `run_agent.py` facade, `agent/turn_*.py`, `agent/conversation_loop.py` |
| Tools | `tools/` (auto-register), `toolsets.py`, `model_tools.py` |
| Terminal backends | `tools/environments/` — local, docker, ssh, modal, daytona, singularity, vercel_sandbox |
| Browser | `tools/browser_tool*.py`, `tools/browser_use_cli.py`, `tools/browser_supervisor.py`, `plugins/browser/` |
| Computer use | `tools/computer_use/` — cua-driver MCP on macOS/Windows/Linux (`website/docs/user-guide/features/computer-use.md`) |
| Skills | `skills/`, `optional-skills/`, `agent/learn_prompt.py`, `agent/curator*.py` |
| Memory | `agent/memory_*.py`, `plugins/memory/` |
| Plugins | `plugins/` (memory, model-providers, image_gen, platforms, kanban, …) + `plugin-catalog/` |
| Session DB | `hermes_state.py` + `hermes_state_*.py` |
| Profiles | `get_hermes_home()`; islands under `~/.hermes/profiles/<name>/` |

**Not on `main`:** `tools/bot_desktop/`, `hermes_cli/web_routers/display.py`, `tui_gateway/methods_display.py`, Desktop `screen-pane.tsx` / `screen-hero.tsx`. Those are **#108914 only**.

### Where “Nous Portal” actually is

**Nous Portal is the billing/OAuth/model+tool gateway, not a Desktop screen.**

| Kind | Path |
|---|---|
| Product docs | `website/docs/integrations/nous-portal.md`, `website/docs/guides/run-hermes-with-nous-portal.md`, `website/docs/user-guide/features/tool-gateway.md` |
| Sidebar | `website/sidebars.ts` → `integrations/nous-portal`, `guides/run-hermes-with-nous-portal` |
| CLI | `hermes_cli/portal_cli.py` (`hermes portal`), `hermes_cli/setup_quick.py` (`hermes setup --portal`) |
| Auth | `hermes_cli/anon_auth.py` (free tier), `hermes_cli/web_server_oauth.py`, `hermes_cli/web_routers/oauth.py` |
| Provider | `plugins/model-providers/nous/__init__.py` (`nous` / `nous-portal`) |
| Host | `https://portal.nousresearch.com` (manage-subscription, models, OAuth) |
| Tool Gateway | paid subscription routes Firecrawl / FAL / OpenAI TTS / Browser Use / optional Modal through Nous (`tool-gateway.md`) |

Desktop copy rule (`apps/desktop/src/AGENTS.md`): free-tier UI must **never** say “Nous Portal” in user-facing text. Hosted dashboard auth still uses Portal OAuth (`web-dashboard.md`).

**Issue #92524** (“Portal/hosted agents — live noVNC of the cloud browser”) is the *hosted* handoff request. #108914 is explicitly the **self-hosted Linux + Desktop** half and **does not close** #92524.

---

## 2. Architecture map (bullets + key paths)

```
Human fronts
  CLI ────────┐
  TUI ────────┤  tui_gateway JSON-RPC  ──► AIAgent (run_agent / agent/turn_*)
  Desktop ────┤                            │
  Dashboard ──┘  (dashboard Chat = TUI PTY)│
                                           ├── tools/registry + toolsets.py
Messaging gateway (gateway/run.py) ────────┤     browser_* / browser_exec
Cron / Bot routines ───────────────────────┤     computer_use (cua-driver)
                                           └── skills, memory, session DB
                                                    │
Nous Portal  (OAuth + Tool Gateway + model catalog)
  hermes portal / setup --portal / plugins/model-providers/nous
```

**Browser stack on `main` (single `cdp_url`):**

1. `BROWSER_CDP_URL` (set by `/browser connect`) — process-global, always wins.  
2. `browser.cdp_url` in config.  
3. Else cloud provider (`browser.cloud_provider`: browser-use, browserbase, firecrawl, camofox, nous).  
4. Else local engine (packaged Chromium via agent-browser, or Lightpanda).  
5. Optional `browser.use_real_profile` snapshot of the user’s Chromium profile.

Resolution: `tools/browser_tool_cdp.py::_get_cdp_override_raw` (no I/O) / `_get_cdp_override` (HTTP `/json/version`). Connect: `tui_gateway/methods_browser.py`, `hermes_cli/cli_commands_mixin.py::_browser_connect`. Driver: default `browser_exec` (`tools/browser_use_cli.py`) or built-in `browser_navigate`… (`tools/browser_tool.py`). `session=` on `browser_exec` already means **own harness daemon + own browser** except when a CDP override pins everyone to one Chrome.

**Computer use on `main`:** `tools/computer_use/tool.py` → cua-driver. Needs a real seat (`DISPLAY` on Linux). Headless Linux **without Bot Screen** has no Xfce/VNC product path on `main`. Camofox already has a *browser* noVNC at `:6080` (`website/docs/user-guide/features/browser.md`) — that is not Bot Screen.

---

## 3. Portal vs Agent vs Desktop boundaries

| Layer | What it is | What it is not |
|---|---|---|
| **Nous Portal** | Subscription + OAuth + Tool Gateway. Lives at `portal.nousresearch.com`. Wired by `hermes portal` / `hermes setup --portal`. | Not the agent process. Not Desktop. Not Bot Screen. Not a CDP endpoint. |
| **Hermes Agent** | The Python core: turns, tools, skills, memory, gateway. `HERMES_HOME` / profiles. | Does not require Portal; Portal is the recommended provider. |
| **Hermes Desktop** | Electron UI over `hermes serve`. Bot Mode = UI over **profiles**. In-app browser = right-pane preview (`apps/desktop` preview/annotate), **not** the bot’s X11. | Not the dashboard SPA. `HERMES_DESKTOP=1` means “this backend was spawned by the app,” not “a GUI is watching” (root `AGENTS.md`). |
| **Web dashboard** | Machine-level admin. Hosted-mode auth can be Portal OAuth. | Chat tab is TUI-in-PTY. No Bot Screen pane on `main`. |
| **Bot Mode** | Roster of named profiles; canonical “Bot Chat”; routines = cron. `website/docs/user-guide/bot-mode.md`, `apps/desktop/src/plugins/hermes-bots/` | On `main` there is **no** “Open Screen” / Screen hero. Those files appear only on #108914. |

---

## 4. Bot Screen #108914 — what exists, where

**State (2026-09-16):** OPEN, **CONFLICTING** vs `main`, ~100 files / +9249, head `hermes/hermes-b802e898`, author teknium1. Related: #92524, #108592, #97859 (earlier Bot Desktop), #17258, #90380, #90374.

**Not in this checkout.** `rg bot_desktop` / `rg CapSolver` / `rg auto_start` on `main` → **no matches**.

### What the PR actually adds (from PR body + file list + overlapping patches)

| Piece | Path (on the PR, not `main`) |
|---|---|
| Xvnc + Xfce launcher | `tools/bot_desktop/launcher.sh`, `runtime.py` — one display per profile under `<HERMES_HOME>/bot-desktop/` |
| Lease | `tools/bot_desktop/lease.py` — `lease.json` + fcntl; agent **or** one human viewer; `human_has_control` |
| RFB filter | `tools/bot_desktop/rfb_filter.py` — drop Key/Pointer/cut-text unless holder |
| Shared Chrome identity | `tools/bot_desktop/browser.py` — dock Browser + agent Chromium share `bot-desktop/browser-profile` |
| Display WS | `hermes_cli/web_routers/display.py` — `/api/display/ws`, one-shot ticket |
| RPC | `tui_gateway/methods_display.py` — `display.status\|start\|stop\|observe\|lease.*` |
| CLI | `hermes computer-use screen status\|start\|stop\|install` |
| Config | `bot_desktop.auto_start` default **false** (`config_defaults.py` on the PR) |
| Desktop UI | `apps/desktop/src/plugins/hermes-bots/screen-*.tsx` — Scheduled Jobs **hero**, Sessions sidebar Screen row, Bots → Open Screen, noVNC Take over / Hand back |
| Docs | `website/docs/user-guide/features/bot-screen.md` |

**Takeover model (round 6):** human-initiated only. Agent-side `request_handoff` / `wait_for_human` **removed**. Bot asks in chat; human clicks Take over.

**Fence vs browser (this is the load-bearing fact for #112937):**

From #108914’s patch on `tools/browser_tool_session.py`:

- `run_fenced` wraps local Bot Desktop browser actions; human lease → `code: human_has_control`.
- `_shares_bot_desktop_browser`: **only** sessions with `features.local` **and** (published `DISPLAY` or human holds).
- Quote from that patch: *“Cloud / user-supplied CDP sessions are a different browser and run unfenced.”*

CDP attach sessions on `main` are created as `_session_record("cdp", …, {"cdp_override": True})` — **not** `features.local` (`tools/browser_tool_session.py::_create_cdp_session`). So CapSolver / `browser.cdp_url` **would stay unfenced** after Bot Screen merges, by design.

Follow-up **#112407** (`fix(bot-screen): fence browser_exec behind human lease`) extends that fence to `browser_exec` for **local** Bot Desktop browsers and again *keeps cloud and user-supplied CDP independent of the lease*.

**OS:** Linux host only. macOS/Windows: `display.status` → `supported: false` (they already have a real seat).

**Swarm:** many open `fix(bot-screen):` PRs (#112849, #112282, #109504, #109544, #112736, …). The feature is still landing, not settled on `main`.

**CapSolver / user CDP vs Bot Screen:** #108914’s dock Browser is the **bot’s** Chromium on the Xfce DISPLAY (shared profile with the agent). It is **not** ADA’s stay-put CapSolver on Xvfb `:99` / `127.0.0.1:9222`. Putting CapSolver on that DISPLAY would mix human RFB input, lease fencing, and a singleton Chrome lock. The #112937 docs are right that v1 should not move it — but that is an **ops rule**, not something either PR encodes.

---

## 5. Teach a task / Screen — shipped vs unmerged

**Shipped on `main` (`/learn`, not a Screen):**

- Slash: `hermes_cli/commands.py` `CommandDef("learn", …)`; handler `hermes_cli/cli_commands_mixin.py` queues `build_learn_prompt()` as a normal turn.
- Prompt: `agent/learn_prompt.py` — no extra model tool; agent authors `SKILL.md` via `skill_manage`.
- Gateway: `gateway/run_inbound.py`; TUI: `tui_gateway/methods_tools.py` `_cmd_learn`.
- Docs: `website/docs/user-guide/features/skills.md` § “Learning a skill from sources (`/learn`)”.
- Dashboard: `web/src/pages/SkillsPage.tsx` **“Learn a skill”** button → composes `/learn` into chat (`ChatPage.tsx` `?learn=`).
- Journey: `/journey` aliases `/learning`, `/memory-graph` — Star Map of learned skills (`agent/learning_graph.py`, `hermes_cli/web_routers/status.py` `/api/learning/*`).

**Desktop on `main`:** no “Teach a task” pane. Closest copy is the tip **“Teach it once”** (`apps/desktop/src/i18n/en.ts`) — skills folders. Skills hub picker is `apps/desktop/src/plugins/hermes-bots/skills-hub.tsx` (install from docs hub). `/learn` still works as a slash because desktop dispatches non-local commands to the backend.

**“Screen” on `main` ≠ Bot Screen:**

- Right-pane **in-app browser** (preview/annotate) — `apps/desktop` `preview-*`, desktop.md.
- Camofox noVNC `:6080` — browser anti-detect, not a bot desktop.
- Bot Screen hero / Open Screen — **#108914 only**.

There is no unmerged “Teach a task” Desktop affordance in the PRs this survey pulled. Do not conflate `/learn` with Bot Screen.

---

## 6. Browser / CDP architecture on `main`

**Single unnamed CDP.** `DEFAULT_CONFIG["browser"]["cdp_url"] = ""` (`hermes_cli/config_defaults.py`). No `cdp_endpoints`, no `browser.profiles`.

**Resolve order today** (`tools/browser_tool_cdp.py::_get_cdp_override_raw`):

1. `BROWSER_CDP_URL`  
2. `browser.cdp_url`

`/browser status` must not HTTP-probe (`tui_gateway/methods_browser.py::_resolve_browser_cdp_url`). Discovery (`/json/version` → `webSocketDebuggerUrl`) only on connect.

**`session=` today** (`website/docs/user-guide/features/browser.md`): each name gets its own Browser Use daemon and, on local/cloud, its own browser. **With a CDP override, every name still hits that one Chrome** — “own tab on that one Chrome.” That is the gap #112937 and #49691 both try to close, in different APIs.

**Playwright vs CDP:** optional-skill `har_capture_cdp.py` uses `playwright.chromium.connect_over_cdp`. Hermes itself drives via agent-browser / Browser Use CLI over CDP, not a second Playwright launch. Launching a second persistent context on the same `user-data-dir` / port is the stay-put footgun #112937 documents.

**Private browser sentinel:** `tools/browser_use_cli.py` `_PRIVATE_BROWSER_SENTINEL` — skip own-tab preamble when the session owns an exclusive Chromium.

**Fences with Bot Screen lease:** **none on `main`.** After #108914, local DISPLAY Chromium is fenced; **user CDP is not**.

---

## 7. Our contribution #112937 — file-by-file

**Upstream:** https://github.com/NousResearch/hermes-agent/pull/112937  
**Head on this fork:** `origin/feat/cdp-endpoints-session-map` (`293a63001e`)  
**vs `main`:** 13 files, +459 / −20, **MERGEABLE**, no reviews/comments yet.

| File | What it changes |
|---|---|
| `tools/browser_tool_cdp.py` | Map parse; resolve order: `BROWSER_CDP_URL` → named map (`endpoint=` / `BROWSER_CDP_ENDPOINT` / `HERMES_SESSION_ID`/`KEY` last segment) → `cdp_url`. `expand_cdp_connect_target`. Signature of `_get_cdp_override[_raw](endpoint=)` with TypeError fallback for zero-arg test stubs. |
| `tools/browser_use_cli.py` | `session=` passed as `endpoint`; distinct URL sets private-browser sentinel; same URL as default stays shared. |
| `tools/browser_tool_session.py` | `bu-named-*` cache keys strip to endpoint name for `_get_cdp_override(endpoint=…)`. |
| `hermes_cli/cli_commands_mixin.py` | `/browser connect <name>` expands map aliases. |
| `tui_gateway/methods_browser.py` | Same expand on RPC connect; status uses `_get_cdp_override_raw()`. |
| `hermes_cli/config_defaults.py` | `"cdp_endpoints": {}`. |
| `cli-config.yaml.example` | Commented CapSolver + lab2/lab3 map. |
| `tests/tools/test_browser_cdp_endpoints.py` | Config/env resolve-order only. **No live Chrome, no Bot Screen.** |
| Docs | `website/docs/user-guide/features/cdp-endpoints.md` (new), plus configuration.md, browser.md, environment-variables.md, `sidebars.ts`. |

**What it does not change:** Desktop, `bot_desktop`, `computer_use`, lease, display RPC, `browser_navigate(profile=)`, fail-loud unknown names.

**New env:** `BROWSER_CDP_ENDPOINT` (behavioral, not a secret). Root rubric hates new `HERMES_*` for non-secrets; this isn’t `HERMES_*`, but it is still an env knob for something that already has config.

---

## 8. Complementarity matrix

| Concern | `main` | #112937 CDP map | #108914 Bot Screen | #49691 / #49693 |
|---|---|---|---|---|
| One stay-put Chrome (`cdp_url`) | Yes | Keeps unnamed default | Unrelated DISPLAY Chrome | `profiles["default"]` falls back to `cdp_url` |
| Extra named Chromes | No | `cdp_endpoints` + `session=` / `/browser connect <name>` | Per-profile Xfce seat, not a CDP map | `browser.profiles` + `browser_navigate(profile=)` |
| Desktop Screen / Start / noVNC | No | Explicitly none | Yes (hero, sidebar, Open Screen) | No |
| Human lease / RFB | No | No | Yes | No |
| Fence CapSolver CDP on takeover | No | Docs *say* future fence; **code does not** | **Unfences** user CDP by design | n/a |
| Unknown name | n/a | **Silent fallthrough to `cdp_url`** | n/a | #49693: **fail-loud** (no silent account cross) |
| Parallel same-Chrome tabs | Own tab, no per-endpoint lock | Still no lock | n/a | Per-endpoint lock + owned tabs |
| Config key | `cdp_url` | `cdp_endpoints` | `bot_desktop.*` | `browser.profiles` |
| Shared files with #112937 | — | — | `config_defaults.py`, `browser_tool_session.py`, `sidebars.ts` | `config_defaults.py` |
| Metal / ADA lab | n/a | Docs assume CapSolver+Xvfb+Tailscale | Xfce+TigerVNC; heavy per bot | Generic |

**#49691 is the sibling, not #108914.** Two PRs adding a name→CDP map with different names and different tool args will confuse reviewers and configs if both land.

---

## 9. Gaps / risks (where #112937 under-read the repo)

1. **Bot Screen fence polarity.** Docs imply “lease-fence loopback CDP when a human holds Screen.” Production Bot Screen code will **not** fence `cdp_override` sessions. Claiming complementarity without quoting `_shares_bot_desktop_browser` is the shallow part.

2. **#49691 / #49693 under-cited in code.** PR body says “does not duplicate”; the *mechanism* is the same category. Silent unknown-name fallback **contradicts** #49693’s fail-loud isolation story.

3. **`session=` overload.** On `main`, `session=` already means isolated Browser Use daemons. Mapping some names onto CDP URLs and falling back to the CapSolver Chrome for unknown names is a silent identity mix — the failure #49693 exists to prevent.

4. **`HERMES_SESSION_ID` / last-segment auto-bind.** Magic. A conversation whose id/key tail matches `lab2` silently leaves the default Chrome. UUID-ish tails are skipped; human-readable names are not.

5. **`BROWSER_CDP_URL` still process-global.** `/browser connect` pins **every** session. Named sidecars need that env unset. Easy to break in Desktop/`serve` (one process, many chats).

6. **Supervisor / unnamed path.** `_ensure_cdp_supervisor` on `main` still calls `_get_cdp_override()` with no endpoint. Named `browser_exec` is wired via `bu-named-*`; built-in `browser_navigate` has no `session=`/`profile=` (that’s #49691). Mixed-driver setups can attach the supervisor to the wrong Chrome.

7. **Tests are change-adjacent, not E2E.** Resolve-order mocks only. Root rubric wants real I/O for config/resolution. No two-Chrome test, no “unknown name must not touch CapSolver cookies,” no Bot Screen lease interaction (correct that Bot Screen isn’t on `main`, but then don’t claim a fence).

8. **Merge conflict surface with #108914.** Both wrap/extend `_run_browser_command` / `_create_session_for_key` in `browser_tool_session.py`. #108914 inserts `run_fenced` around the unfenced body. #112937 only changes `_create_session_for_key` CDP lookup. Mechanically mergeable if #112937 stays in `_create_session_for_key` and does not rewrite `_run_browser_command` — but #108914 is already conflicting vs `main`, so the three-way will be messy.

9. **Public docs leak ADA lab.** CapSolver, Xvfb `:99`, Tailscale `:6080`, “ADA-VPS MacBookPro9,2” in a user-guide page is operator runbook, not product. Reviewers may bounce it or ask it to move out of `website/docs`.

10. **No `hermes tools` / setup UX.** New config map is YAML + env only. Project taste is `hermes tools` / setup, not raw keys.

11. **TypeError stubs.** Production carrying `except TypeError: zero-arg _get_cdp_override()` is a test-compat wart.

12. **CapSolver is not in-tree.** Zero references on `main`. It is an ADA sidecar, not a Hermes feature. Tying the PR narrative to it overfits one host.

---

## 10. Concrete follow-ups for Ray

**On the #112937 review comment / PR body**

1. **Keep:** “Not a Desktop Screen / Start button; not display spawn.” That is true and useful next to #108914.
2. **Rewrite the complement sentence.** Accurate version:  
   *#112937 is orthogonal to Bot Screen’s Xfce+RFB seat. Bot Screen fences the **local DISPLAY** Chromium; user-supplied CDP (`cdp_url` / `cdp_endpoints`) stays unfenced by #108914/#112407. Do not put the stay-put CapSolver Chrome on the Bot Screen DISPLAY — that is an ops rule, not implemented in either PR.*
3. **Drop or demote** “lease-fence local/loopback CDP when a human holds Bot Screen” unless you are volunteering a **follow-up** that *changes* Bot Screen’s explicit unfence — which fights that PR’s design.
4. **Be honest about #49691:** same problem class (name → CDP). If both might merge, pick one key (`profiles` vs `cdp_endpoints`) or document a migration. Silent fallback vs fail-loud is a product decision; don’t pretend they don’t interact.
5. **If this is a general Hermes PR:** strip ADA/CapSolver/Tailscale metal from `cdp-endpoints.md`; keep a private runbook in the companion repo. If this is ADA-only, say so so upstream can close as out-of-tree.

**On the patch (only if you choose to change #112937)**

- Fail-loud on unknown `session=` map names **or** document loudly that unknown names share CapSolver cookies.
- Do not auto-bind `HERMES_SESSION_ID` unless that is a stated ADA requirement; it is the sharpest footgun.
- Add 1–2 invariant tests: distinct map URL ⇒ distinct override; unknown name ⇒ either error or *documented* default; `BROWSER_CDP_URL` still wins.
- Leave `browser_tool_session._run_browser_command` alone so Bot Screen’s `run_fenced` can land.
- Avoid a second behavioral env var if a reviewer will quote the rubric; `session=` + config map is enough.

**Do not expand into Bot Screen, Teach-a-task UI, or a Screen button.** That was the right restraint. The miss was citing Bot Screen as if the lease model had been read.

---

## Sources (this pass)

- Tree: `origin/main` `03b0c79472`; branch `origin/feat/cdp-endpoints-session-map` `293a63001e`
- PRs: [NousResearch/hermes-agent#112937](https://github.com/NousResearch/hermes-agent/pull/112937), [#108914](https://github.com/NousResearch/hermes-agent/pull/108914), [#49691](https://github.com/NousResearch/hermes-agent/pull/49691), [#112407](https://github.com/NousResearch/hermes-agent/pull/112407)
- Issues: [#49693](https://github.com/NousResearch/hermes-agent/issues/49693), [#92524](https://github.com/NousResearch/hermes-agent/issues/92524)
- Files read in-tree include: `AGENTS.md`, `tools/browser_tool_cdp.py`, `tools/browser_tool_session.py`, `tools/browser_use_cli.py`, `tui_gateway/methods_browser.py`, `hermes_cli/config_defaults.py`, `hermes_cli/portal_cli.py`, `agent/learn_prompt.py`, `apps/desktop/AGENTS.md`, `apps/desktop/src/AGENTS.md`, `website/docs/integrations/nous-portal.md`, `website/docs/user-guide/{desktop,bot-mode,features/skills,features/browser,features/computer-use,features/web-dashboard}.md`, `website/sidebars.ts`
- #108914 patches inspected for `tools/browser_tool_session.py`, `tools/browser_tool.py`, `hermes_cli/config_defaults.py`
