# Maintaining the user manual

How `docs/user-manual/` is kept current: analyse what changed, stage a live
instance, capture screenshots, update the markdown, regenerate the HTML, ship a
PR. Modelled on Airlock's process (`airlock-manual/manual-maintenance.md`),
with the differences that matter here called out — chiefly that most of what a
RemotePlayCore screenshot shows is **discovered from the network**, not seeded
over REST.

**The artifacts**

| File | Role |
|---|---|
| `docs/user-manual/README.md` | The manual — single source of truth. Edit this. |
| `docs/user-manual/remoteplay-manual.html` | Generated. Never hand-edit; rebuild with `node build/build-manual.mjs`. |
| `docs/user-manual/img/*.png` | Screenshots, numbered `NN-slug.png`, referenced relatively. |
| `build/build-manual.mjs` | Markdown → branded HTML (nested sidebar, Remote Play green ramp, light + dark); **fails on broken in-page anchors**. |
| `build/manual-screenshots.mjs` | Playwright/Edge capture against a live instance, with the simulators supplying the plant. |
| `.github/workflows/publish-manual.yml` | Mirrors the **committed** `docs/` tree to the public `remoteplaycore-manual` repo on merge to main. It does not regenerate — a stale committed HTML ships stale. |

The manual's footer stamp — *"generated against main @ `<sha>`, RPC-1…`<n>`"* —
is the baseline for every update. Bump it every cycle.

---

## Phase 1 — Discover what changed

1. **Baseline** = the sha in the manual's footer stamp.
2. `git log <baseline>..origin/main --oneline` — separate user-visible work
   (SPA, endpoints, operational behaviour) from internal churn. Commits name
   their tickets (`(RPC-nn)`).
3. **Jira**: `project=RPC AND key >= RPC-nnn ORDER BY key`, plus older tickets
   whose status moved. Trust **merged code over ticket status** — document what
   is on main and only what is on main.
4. **Deep diff**: `git diff <baseline>..origin/main -- web/RemotePlayCore.Web/src src/RemotePlayCore.Control docs`
   and produce, per feature: **verbatim UI labels** (the manual quotes the UI
   exactly — paraphrased labels are bugs), **manual section impacts**,
   **screenshot triage** (every `img/NN-*.png` as KEEP / RETAKE / NEW), and the
   **REST surface** changes, because the seeding in `manual-screenshots.mjs`
   breaks on removed fields and new server-side validation.
5. Labels live in `web/RemotePlayCore.Web/src/locales/en.json`, not in the
   components — read them there rather than transcribing from a screenshot.
   Backend responses stay English and the client maps codes, so a message in
   the UI may not exist as a string anywhere in the server.

## Phase 2 — Environment

1. **Worktree, never the developer's checkout**:
   `git worktree add $TEMP/rpc-wtN -b docs/user-manual-update-N origin/main`.
2. **Build**: `./build/build-web.sh` then `dotnet build`.
3. **Run** with a disposable data directory, on a port nothing else owns:
   ```
   dotnet run --project src/RemotePlayCore.Control --no-build -- \
     --urls http://127.0.0.1:5788 --DataDir <fresh dir>
   ```
   A fresh `--DataDir` boots with the bootstrap **admin** / **admin** account
   (logged with a warning). Check the port is free first, and never point the
   seeding at a live instance — it authenticates as admin and writes.
4. **The instance must outlive the shell that started it.** Started as a
   backgrounded child of a short-lived shell it dies between steps and the
   capture run fails with *"timeout waiting for server"*. Start it detached.

### The network is part of the fixture — isolate it

This is the RemotePlayCore-specific hazard, and it has already bitten: a
capture run on a developer machine picked up **33 devices and 116 sources**
when the simulators were only advertising 11 devices. The rest were real gear
on the office LAN, with real hostnames and real IP addresses, and they went
straight into an image destined for a public manual.

Before a shoot that will be published, do one of:

- run the instance on a host with only an isolated NIC on the AoIP side, or
- bind discovery to an interface nothing else advertises on
  (`--Livewire:BindAddress=<ip>`), or
- review every captured image for third-party device names and addresses and
  retake on a clean network — not crop, because the counts in the header give
  it away too.

## Phase 3 — Seed the plant

`build/manual-screenshots.mjs` does this automatically. What it sets up:

- **The simulators**, spawned as child processes and killed on exit:
  `RemotePlayCore.LivewireSim` (6 studios × 4 sources, audio on the first four)
  and `RemotePlayCore.Aes67Sim` (5 devices × 2 streams, one vendor profile per
  device, so the Sources tree shows the **AES67** badge beside Dante-, Ravenna-
  and Axia-shaped names). Both take `--seed`, and the seed is fixed: a retake
  reproduces the same fleet, which matters because the prose names specific
  studios. `--no-sim` expects them to be running already.
- **Discovery gate**: the script waits for `/api/sources` to report the
  expected device count before shooting. An empty-state Sources tree is the
  classic stale-fixture failure and it will otherwise pass silently.
- **A source list** (`On-air studios`) so the sidebar shows a list entry and
  the Source Lists page is not empty.
- **A panel** (`Studio monitoring`) with its first row of pads assigned from
  live discovery. An all-**ASSIGN** panel is a correct screenshot of an empty
  panel and tells the reader nothing.
- **A second, non-admin account** so the Users page shows the two-role model.

## Phase 4 — Screenshots (Playwright + installed Edge)

```
node build/manual-screenshots.mjs --base http://127.0.0.1:5788 [--only 03,08] [--no-sim]
```

Playwright driving the installed **msedge** channel (`--channel`), so there is
no bundled-Chromium download: viewport **1600×1000 @ deviceScaleFactor 1.5**,
dark colour scheme. Every shot is independent — a failure keeps the previous
image and is listed in `manual-screenshots-summary.json`.

**Selector discipline** (each of these was learned here, not inherited):

- The SPA has **no test ids**. Click by visible text with an
  `offsetParent !== null` filter.
- **Duplicate text is the main hazard.** "Sources" is both a sidebar entry and
  the panel's drawer toggle; matching by text alone picks the sidebar entry and
  silently navigates away, so the "source drawer" shot becomes a second copy of
  the Sources page. Scope the click to the surrounding toolbar — find a sibling
  unique to that strip (**SETTINGS**) and search within its parent.
- The **username field has no `type` attribute**, so `input[type=text]` never
  matches it, and it has no name or id either. Its lowercased placeholder
  (`input[placeholder="username"]`) is the only stable handle.
- Native `confirm()` dialogs cannot be screenshotted; auto-accept them via
  `page.on('dialog')` and quote the text in prose.
- Give discovery and meters settle time. Advertisement periods are seconds
  apart; a shot taken too early shows a half-populated tree.

## Phase 5 — Write the markdown

- **Framing**: Remote Play is a gateway for **Livewire and AES67** — the two
  are peers. Never present AES67 as an add-on.
- **Voice**: bold **verbatim UI labels**, explain the operational *why*
  alongside the *what*, tables for enumerable facts, prose for behaviour. Quote
  banner and dialog text exactly.
- **Depth follows operator risk**: the licence lifecycle, role model and
  persistent state get full subsections; cosmetic fixes get nothing.
- **Structure**: two-level Contents; cross-reference by anchor. Anything an
  operator would scan for must appear in the Contents.
- Reconcile, don't append — new features usually contradict old sentences.
- Bump the footer stamp: `main @ <sha>`, `RPC-1…<n>`.

## Phase 6 — Generate and validate

```
node build/build-manual.mjs
```

regenerates `remoteplay-manual.html` and **fails if any in-page anchor doesn't
resolve** — heading renames get caught here. Also check every `img/...`
reference exists on disk, and open the file in a browser in both light and dark
to confirm the brand ramp holds (the green shifts a full step by surface —
`#22c55e` is 2.3:1 on white and is never used as text there).

## Phase 7 — Ship

1. Commit README + HTML + images together on the `docs/user-manual-update-N`
   branch; PR to main with the ticket reference. The PR body lists coverage per
   ticket and any product inconsistencies found along the way — manual work
   regularly surfaces real bugs; file or flag them, don't write around them.
2. On merge, `publish-manual.yml` mirrors `docs/` to the public
   `remoteplaycore-manual` repo and its Pages site.
3. Clean up: stop the instance, remove the worktree and the disposable data
   directory, and leave the developer's working tree exactly as found.
