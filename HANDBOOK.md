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

## Architecture

- **One file.** All CSS/JS modules inline in `index.html` (~500 KB). Deploy = replace the file in this repo
  (drag-drop upload on github.com → commit) → GitHub Pages updates → hard-refresh devices (⌘⇧R).
- **Modules** (in order): CORE (state/sync/inbox engine) → UIKIT (h(), popovers, modals, menus) →
  XLSX (dependency-free .xlsx writer) → THEME/STORE → SHELL (nav) → PALETTE (⌘K) → views
  (WEEK, NOTES incl. editor, OUTREACH, RADAR, ASSISTANT, SETTINGS) → BOARD engine → boot.
- **No build step, no dependencies, ES5-style code.** Test rig: jsdom (see Testing).

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
   volume stress. Editor selection APIs work in jsdom; `execCommand` does not (fallback paths cover it).
3. After deploy: hard-refresh every device (each browser caches the old file).

## Handover ritual for a new chat/session

1. Clone this repo, read this file, read `index.html` selectively (grep module markers).
2. Never create a new Gist for the user (splits their cloud) — the app's Settings guard warns about this.
3. Routine changes: edit the scheduled task's prompt (Cowork → Scheduled), not the app.
4. Ship: updated `index.html` (and this handbook if the system changed) → user drag-drops to repo → verify live.
