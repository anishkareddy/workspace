# Workspace — System Handbook

This repo hosts a single-file personal workspace app (`index.html`) served via GitHub Pages.
This handbook exists so any future Claude session (or human) can pick up development or operations
with zero prior context. **Start every new dev session by reading this file.**

## What this is

A private workspace for a VC investor (UK VCT, invests at Series A/B):

- **Week** — weekly task tracker (4 lanes, day groups, carry-forward)
- **Notes** — notebooks → sections → pages; rich editor (tables, callouts, code, toggles, boards)
- **Outreach** — investor-networking CRM: This week / Meetings / Contacts / Companies tabs.
  **Single-entry model: the meeting is the source of truth.** Logging a meeting auto-creates or
  updates the contact, stamps lastContactDate, can set the contact's next step, and the contact's
  deal history (table column, tooltip, Excel) is DERIVED from its meetings — contact-level
  dealsShared/dealsReceived fields are legacy-read-only (still counted, never edited via UI).
  Contact modal = 8 essentials + collapsible "More details" + inline meeting history with a
  prefilled "Log meeting" shortcut. One-click "Copy for Affinity" per meeting; Excel button is
  tab-contextual (contacts vs meetings).
- **Deal Radar** — UK+EU pre-seed/seed tracker (Fintech · AI) with three tabs:
  - *Deals* — the radar table (sector, round, amount, investors, UK-nexus tag in "why")
  - *Market pulse* — auto-computed trends board + twice-weekly pulse notes (market diary)
  - *Theses* — quality-bar investment theses
- **Reads** — the reading room: twice-weekly (Tue+Fri) long-form briefings + deep dives written by
  the `deep-read-briefing` routine to her "Daily VC Briefing" spec (five domains — fintech,
  healthtech, European VC, AI/deeptech, macro — TL;DR, Worth Reading, Thesis Radar; Fridays add
  podcast picks + weekly thesis tracker). Cards mark read on open; pin to exempt from the 30-item
  cap; "Save to Notes" files a read permanently into Notes → Reference.
- **Assistant** — in-app Claude chat (optional; user's own API key, never synced)

### Dossiers (v9.6): read/edit vs create

Outreach and Deal Radar follow a two-surface pattern:

- **Forms create** — "New deal" / "Add contact" / "Log meeting" open the classic field modals.
  Row **pencil** buttons and the dossier's "Edit fields" button reopen them (escape hatch).
- **Dossiers read & edit** — clicking any row (radar deal, contact, meeting, This-week item,
  palette jump) opens a `modal.dossier`: big serif title, one-click **status chips**, a facts
  grid (dates/links), and full-width sections rendered through mdLite. Every section is
  **inline-editable**: click text → auto-sizing textarea → Enter/⌘Enter or blur saves,
  Escape reverts (and never closes the modal). Deal dossiers have **←/→ prev-next** over the
  current filter (keys ignored while typing) and a Watch toggle; contact dossiers embed the
  derived deals list + meetings timeline (per-meeting Affinity copy, chevron into the meeting
  dossier, prefilled "Log meeting"); meeting dossiers show shared/received deal columns and a
  prominent "Copy for Affinity". Editing a deal's "why" preserves its trailing `[nexus]` tag.
  Shared components live in UIKIT as `W.de` (edit/chips/sec/fact/date).

## Architecture

- **One file.** All CSS/JS modules inline in `index.html` (~540 KB). Deploy = replace the file in this repo
  (drag-drop upload on github.com → commit) → GitHub Pages updates → hard-refresh devices (⌘⇧R).
- **Modules** (in order): CORE (state/sync/inbox engine) → UIKIT (h(), popovers, modals, menus) →
  XLSX (dependency-free .xlsx writer) → THEME/STORE → SHELL (nav) → PALETTE (⌘K) → views
  (WEEK, NOTES incl. editor, OUTREACH, RADAR, ASSISTANT, SETTINGS) → BOARD engine → boot.
- **No build step, no dependencies, ES5-style code.** Test rig: jsdom (see Testing).

### v10 "Atelier" design system (visual layer only — zero behavioural change)

The v10 redesign rebuilt the CSS foundations to Apple-grade materials while keeping every
selector, aria-label and API stable. Know these before touching styles:

- **Tokens** (`:root` + `[data-theme=dark]`): glass recipes `--glass-chrome/panel/overlay` +
  `--glass-blur(-lg)`; top-edge highlights `--hl`, `--hl-soft` (the "lit from above" tell, used
  as `inset 0 1px 0`); elevated overlay surface `--bg-elevated` (lighter than surfaces in dark);
  5-step shadow ramp `--shadow-xs…xl` with legacy aliases (`--shadow-card`,
  `--shadow-card-hover`, `--shadow-popover`, `--shadow-modal`) that all views still read;
  radii scale r-xs 5 → r-xl 18 (modals 18, cards 12, controls 8); easings add `--ease-spring`
  (subtle overshoot) and `--ease-glide`. Serif stack leads with Iowan Old Style (macOS-shipped;
  NO external fonts — offline integrity). Version marker: `<html data-app="10.0">` (used for
  live-deploy verification) + the About card string.
- **Glass surfaces** (backdrop-filter budget — keep bounded): sidebar rail gradient, view
  headers, editor/board toolbars, sticky table headers (`.ot-hd`, `.rd-hd`), palette, modals,
  popovers, toasts, tabbar, assistant composer. Never put filters on rows.
- **Segmented control** (`W.segmented`, UIKIT): same API; now renders a decorative
  `span.seg-thumb` that slides via transform (positioned with offsetLeft/Width on rAF + click).
  When unmeasured (jsdom, first paint) the `.seg` lacks `[data-thumb]` and a CSS fallback paints
  the pressed button directly — aria-pressed remains the truth, tests unaffected.
- **Sidebar indicator** (SHELL): `span.sb-ind` inside `.sb-nav`; `updateNav()` positions it via
  `--ind-y` and sets `[data-ind]`; without measurement (jsdom) the static `.sb-item.active`
  fallback rule applies.
- **Staggered entrance** (SHELL render): `#view` gets `.view-enter` for 600 ms ONLY when the
  view actually changes (never on same-view re-renders — protects editor/typing paths); CSS
  `rise-in` rules are all scoped under `#view.view-enter`. Reduced-motion kills them.
- **Week large-title condensation**: pure-CSS scroll-driven (`animation-timeline: scroll()`)
  behind `@supports` — Chrome-only enhancement, inert elsewhere and in jsdom.
- Embedded inputs inside composite fields (palette head, rd-search, composer, cmp-box, ai-row)
  explicitly kill their own focus box-shadow — the container carries the ring.

## Data & sync model

- State lives in `localStorage` (`workspace_state_v3`) and mirrors to a **private GitHub Gist**
  (`workspace.json`; token + Gist ID in Settings → Sync, stored only in each browser).
- **Merge**: per-record LWW by `updated` timestamp (`mergeCollection`); deletes are tombstones
  (`deleted:true`) pruned after 45 days; first sync on a fresh device ADOPTS the cloud (no merge).
- Auto-push ~1.5 s after edits; auto-pull on open and when the tab regains focus.
- **State schema** (collections all follow `{id, updated, deleted?}`):
  `notes.notebooks[{sections[]}]`, `notes.pages[]` (order field = manual sort),
  `tracker.tasks[]`, `tracker.dayNotes{}`, `outreach.companies[]`,
  `outreach.contacts[]` (incl. `summary`, `dealsShared`, `dealsReceived`),
  `radar.deals[]`, `radar.theses[]`, `radar.pulses[]` (capped at 60 live),
  `reads.items[]` (briefings/deep dives; 30 live unpinned cap; `readAt`, `pinned`, `kind`),
  `ai.chats[]`, `processedInbox[]` (capped ~500).
- **mdLite** (`CORE.mdLite`): converts markdown-ish plain text (### headings, - bullets, **bold**,
  links, ---) to app HTML. Used by applyCommand for note/thesis/pulse/read bodies; migrate() also
  repairs any legacy records that stored literal "### " markdown inside paragraphs.

## The inbox contract (how anything external writes in)

On every sync the app reads `inbox.json` from the same Gist: an array of commands
`{id:"unique", type:"…", …}`. Processed ids are remembered; unknown types are left for newer app versions.
Command types (see `applyCommand` in CORE):

- `task` {task, lane, day, company, week…}
- `contact` {name, firm, title, email…}
- `note` {title, content|html, section…}
- `complete_task` {match}
- `deal` {company, sector, round, amount, investors, announced, source, summary, why} — dedupes on company+round (enriches instead)
- `thesis` {title, content|html, tags[]} — dedupes on title
- `pulse` {title, content|html, date} — dedupes on title+date, cap 60 live
- `read` / `briefing` {title, content|html, date, kind:"briefing"|"deepdive", tags[]} — dedupes on
  title+date; cap 30 live unpinned (pinned exempt); markdown bodies rendered via mdLite
- `meeting` {name, firm, date, kind, dealsShared, dealsReceived, summary, notes} — dedupes on
  name+date; auto-links or creates the contact and advances its lastContactDate

Scheduled routines skip `inbox.json` and instead call `window.__WS.applyCommand(cmd)` directly in a
browser tab, then `window.__ws.eng.persistLocal(); window.__ws.eng.push();` — same contract, no token handling.

## Automations (Cowork scheduled tasks on the user's machine)

- **`uk-deal-radar-scan`** — Mon + Thu 08:00 local. Unified investor radar, built as a
  **multi-agent workflow**: a thin orchestrator (session default model) spawns four parallel
  SONNET gatherers via the Agent tool (Harmonic / PitchBook / press / Gmail newsletters), each
  returning strict compact JSON; the orchestrator merges + dedupes, does watchlist/warm-path/
  thesis-cohort enrichment, then one OPUS synthesis agent ranks deals, writes the pulse, applies
  the thesis quality bar and drafts the digest; finally the orchestrator pushes via the browser.
  Failure-isolated (a dead gatherer is skipped), context-isolated (700KB Harmonic dumps never
  reach the synthesis layer). Five sources:
  1. *Harmonic* (team instance — READ-ONLY on shared assets): natural-language discovery searches;
     net-new from team saved searches (UK/EU Pre-Seed Weekly 158263, UK Recently Raised Seed 157352,
     UK High-Qual Stealth 156637, High-Traction Seed B2B SaaS 154457, Strong-Founder UK B2B 144491);
     people signals (AI leaders in fintech 144493, operators who left unicorns 158268);
     writes ONLY to her list `urn:harmonic:company_watchlist:4fbc9646-c009-4f4e-8136-2ebb2eb189ed`.
     Gotchas: `get_saved_search_results` responses are huge (use net-new → `get_companies` with minimal
     field_groups); team search names can mislead — always gate by headcount/stage/geo.
  2. *PitchBook* fund sweep (PBIDs: Concept 226163-53, Crane 187432-21, Seedcamp 51064-84,
     LocalGlobe/Phoenix Court 148770-82, Playfair 55725-31, Ada 267524-47, SFC 60291-64,
     Episode 1 59900-77, Cherry 100624-51, Point Nine 52874-92).
  3. Press sweep (UKTN, Sifted, tech.eu, TechCrunch, EU-Startups, Finextra…).
  4. Gmail newsletter signals (Sifted, Fintech Brainfood, Venture Daily Digest, AI Inner Circle).
  5. Watchlist re-check (app "Watching" deals ↔ Harmonic list + PitchBook rounds).
  6. Team warm paths: `get_company_connections` on shortlisted deals — who at Octopus knows people
     there (team network ≈23.5k connections; never pull contact/correspondence fields).
  7. Thesis validation loop: each active thesis gets a "Thesis — … (auto)" Harmonic cohort list
     (created by the routine, owned by her); runs report cohort movement (raises/headcount) as
     strengthening/flat/weakening evidence in the pulse's "Thesis check".
  Pushes deals + one pulse note (+ thesis when the quality bar is met) into the app via the browser,
  then posts a ≤1-minute digest in chat. The pulse is holistic: alongside early-stage sections it
  carries "Big moves" (mega-rounds, exits/M&A, fund launches — partner-level awareness) and
  "The wider picture" (cross-sector/geopolitical/macro, investor-framed). Mega-deals stay out of
  the deals table except rare strategic UK/EU fintech-AI growth rounds (round:"Growth" — excluded
  from median stats automatically).
  Ad-hoc: any new chat can run "explore a thesis" interactively — natural-language search →
  gated market map → optional "(auto)" list; the Harmonic console is for visual deep-dives only.
- **`deep-read-briefing`** — Tue + Fri 08:00 local. Multi-agent deep-reading workflow built from her
  claude.ai "Daily VC Briefing" project spec: Sonnet gatherers (Tier-1 newsletters read in full —
  Fintech Brainfood, Sifted, Venture Daily Digest, EUVC; Tier-2 weeklies when present — Stratechery,
  Benedict Evans, Exponential View, The Generalist, Fintech Takes, Not Boring, The Diff, CB Insights;
  a 5-domain web sweep; Friday podcast scout) → one Opus writer produces the full briefing (British
  English, opinionated, five domains + TL;DR + Worth Reading + Thesis Radar, Friday podcast picks +
  weekly thesis tracker) plus a separate 600–1,000-word deep dive on the window's strongest signal →
  both pushed as `read` commands. Self-healing lookback from the latest read's date.
- **`wednesday-outreach-brief`** — Wed 08:00, outreach follow-up reminder.
- Runs are **stateless by design**: every run is a fresh session; all durable state lives in the app/Gist.

## VCT mandate (why deals carry a UK-nexus tag)

Qualifying investments need a **UK permanent establishment at time of investment** (not UK incorporation).
Post-Apr-2026 limits: gross assets ≤£30m pre-investment; £10m/yr, £24m lifetime (double for
knowledge-intensive); first risk-finance within ~7 yrs of first commercial sale (10 KIC).
Applied loosely at radar stage — every deal's "why" ends with `[HQ London]` / `[Berlin HQ; UK office]` /
`[no UK nexus yet — watch]` style tags.

## Monthly reviews (v10.1) — the archive compiles itself

Pulses cap at 60 live and reads at 30 unpinned, so the raw archive rotates. v10.1 makes the
durable record automatic: once a month has any pulses/reads/deals, the app compiles
"**<Month> <Year> — in review**" into **Notes → Reviews** (section auto-created; notes pages
are never capped, so these survive forever and are ⌘K-searchable).

- **Auto**: on boot, if last month has material and no review page exists, it compiles the
  structural record silently (deals table + "the ones that mattered" with nexus tags +
  condensed pulses + reads index + thesis movements) and toasts with an Open action.
  Idempotent — dedupes by title, never touches an existing page.
- **Manual**: Deal Radar → Market pulse → **Compile month** (recompile last month · this
  month so far · "Rewrite with AI" when an Assistant API key is set — Claude writes the
  editorial synthesis in British English: themes, thesis evolution, deals that mattered,
  watch-next-month; the structural record is appended below an `<hr>`). AI failure falls
  back to structural, never loses the page.
- Engine in CORE: `monthMaterial(ym)`, `compileMonth(ym,{ai,force})`, `autoRollup()`,
  `findRollup(ym)`, `rollupTitle(ym)`, `openRollup(page)` — all on `ENG`. Manual compile
  REPLACES page content (generated page — hand-edits belong elsewhere); auto never recompiles.
- `applyCommand` `note` now auto-creates a named section that doesn't exist yet (routine
  pushes land where aimed).

## Storage: images are the only real risk

Pasted images live as data-URLs inside page HTML — measured in the wild, two screenshot-heavy pages
held >1 MB of a 1.6 MB state. Mitigations (v9.5): paste-time downscale tightened (≤1200px, q0.8);
Settings → Data → **"Slim images"** re-compresses every embedded image (≤1100px, q0.72, skips
<60 KB images and anything that doesn't shrink ≥10%); a one-time warning toast fires when state
crosses 2.5 MB. `pull()` already handles Gist truncation (>1 MB files read via raw_url).
Assistant chats cap at 40 live (migrate tombstones the oldest).

## Scale guardrails (measured)

Six months of heavy use (≈350 deals + 60 pulses + 25 theses) ≈ **105 KB** of state — ~2% of the
localStorage budget and well inside comfortable Gist size. Guardrails: pulse cap (60 live),
processedInbox cap, 45-day tombstone prune, deals time-window filter (auto-defaults to 90 days past
60 deals). The one thing that genuinely eats space is pasting many images into notes (data URLs) —
prefer links/covers for heavy media.

## Testing (before every deploy)

1. Syntax: extract every `<script>` block, `new Function(src)` each.
2. jsdom suites (polyfill `matchMedia`, `ResizeObserver`, stub `fetch`, `runScripts:"dangerously"`):
   boot/seed, applyCommand types + dedupe, inbox, merge/prune, undo/redo, callout/code Enter+paste,
   tables, reorder, outreach fields, xlsx (unzip + openpyxl), radar (deals/pulse/thesis/trends math),
   dossiers (open from every row type, inline edit save/Escape, chips, prev/next keys, Affinity copy,
   nexus-tag preservation, Edit-fields escape hatch), volume stress. Editor selection APIs work in
   jsdom; `execCommand` does not (fallback paths cover it). NOTE: row click now opens the dossier —
   tests that need the form modal must click the row's pencil (`button[aria-label="Edit …"]`).
   v10 additions: version markers (`data-app`), token completeness both themes, legacy shadow
   aliases, seg-thumb decorative + aria truth, `.sb-ind` presence + static fallback rule,
   `rise-in` scoped only under `#view.view-enter`, contrast spot-checks on token hexes.
   Dossier deal-navigation arrow keys are bound to the modal box, not document — dispatch there.
   Deal dedupe enriches EMPTY fields only; processedInbox ids are recorded by processInbox (the
   inbox.json path), not by direct applyCommand calls.
3. Real-browser visual pass when the sandbox allows: headless Chromium (playwright-core +
   `/opt/pw-browsers` binary) — screenshot every view light+dark+mobile and review.
4. After deploy: hard-refresh every device (each browser caches the old file).

## Handover ritual for a new chat/session

1. Clone this repo, read this file, read `index.html` selectively (grep module markers).
2. Never create a new Gist for the user (splits their cloud) — the app's Settings guard warns about this.
3. Routine changes: edit the scheduled task's prompt (Cowork → Scheduled), not the app.
4. Ship: updated `index.html` (and this handbook if the system changed) → user drag-drops to repo → verify live.
