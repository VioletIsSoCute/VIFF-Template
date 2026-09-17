# VIFF Equipment / TEA Toolkit — Manual & Handoff Doc

*Last updated: 2026-09-10 (Phase 0-5 of the academic-methodology integration: costing-schema
columns, pressure factor, convention-aware bare-module math, and a fifth `plant_economics.html` tab
carrying the FCI/TCI/COM/DCF cascade for Turton, Seider and Towler-Sinnott side by side. See §9.) Written to let a new conversation thread (human or Claude) pick this project up
without re-deriving decisions already made. If you're a future Claude session reading this: the
design-decisions section exists specifically so you don't have to re-ask questions already
answered.*

> **New session starting here?** Read `TODO.md` first (what is not done, and the rules), then
> `HANDOVER.md` → OPEN GAPS (why), then §4 and §6 below. §8 at the end is the orientation note.

## 1. What this is

A browser-based toolkit for techno-economic equipment cost estimation and process-level OpEx
estimation, built for VIFF's standardized TEA/LCA reporting template project. It has four tools
sharing one data layer:

- **Equipment Selector** — a guided decision-tree Q&A that helps someone unfamiliar with the
  equipment library find the right cost correlation for a piece of equipment.
- **CapEx Estimator** — a calculator that takes chosen equipment + sizes and produces installed
  costs, with a running total, cost-breakdown charts, and export.
- **SFF CapEx Matcher** — uploads a PISCES SFF file, fuzzy-matches each unit against the equipment
  library, and walks a human through confirming, overriding, bootstrapping, or manually adding
  an entry for every unit — nothing is added to the library automatically.
- **SFF OpEx Matcher** — uploads the same SFF file and covers OpEx in two modes (a mode
  switch inside the one tool, not two separate tools): **Boundary Streams** pulls out every feed/
  product/waste stream, suggests a price for each from whatever the file itself reports, and walks
  a human through confirming or overriding every one; **Utility Duty & Annual OpEx** (new, see
  §5.5) prices each unit's own steam/cooling/electricity *duty*, then rolls everything — boundary
  streams, utility duty, and a Maintenance/Labor "% of Total Installed Cost" estimate — up into a
  single Annual OpEx figure, pulling that Total Installed Cost live from the Estimator tab.

Since **Round 19** the toolkit opens on a **Home** tab that says what it is, lists the seven steps
of a review in the order they are meant to be done — each naming what it *produces* and carrying a
button that goes there — and explains the SFF files the whole thing reads, quoting Project PISCES
(https://projectpisces.org/). It holds no state, fetches no data file and posts exactly one message
type; it is documentation with working links, not a dashboard. Any tab can also be deep-linked by
hash: `equipment_toolkit.html#econ`, `#sff`, `#estimator`, `#streams`, `#lca`, `#selector`,
`#appendix`, `#home`. An unknown hash falls back to Home.

All four are static HTML files with no build step and no server-side code. They only need to be
served over `http://` (not opened as `file://`) because they `fetch()` shared JSON files. Three
different postMessage bridges run through `equipment_toolkit.html`'s wrapper: an SFF-file broadcast
shared between the SFF CapEx Matcher and SFF OpEx Matcher (see §5.4 and §6), a CEPCI sync from the
Estimator to the SFF CapEx Matcher, and a Total Installed Cost sync from the Estimator to the Streams/OpEx
Matcher (see §5.5). Each tool still works standalone if opened directly (no wrapper, no cross-tab
magic — just upload there too, and the Annual OpEx panel's TIC field becomes manually editable
instead of live-synced).

**As of the 2026-09-10 update the capital and operating-cost gaps listed below are largely closed**
— working capital, indirect/Lang-style capital, depreciation/tax/DCF and consumables all now live in
`plant_economics.html` (§5.6, §9). The paragraph is kept as written because it records what the tool
looked like before, and because two of its caveats still stand: nothing here is validated against a
real published TEA end-to-end, and *which* convention to report remains the user's call.

**This does not yet cover LCA/emissions, and does not yet fetch SFF files live from PISCES** — SFF
files are uploaded by hand as JSON. Both are explicitly future work (see §7). Per-unit utility duty
is now priced (§5.5), but there's still a real gap between what this toolkit produces and a full
published-literature TEA: no working capital, no indirect/Lang-factor-style capital costs (piping,
instrumentation, engineering, contractor fee, contingency — everything beyond the bare-module/
installation factor already in the equipment library), no depreciation/tax/discounted-cash-flow
layer, and no consumables (catalysts, solvents) tracking. See the note at the top of §7 for the
prioritized list — some of this may not even be needed depending on whether the immediate consumer
is the supply-chain/superstructure optimization model (which mainly wants a CAPEX + annual-OpEx
pair) or a standalone paper-style TEA (which usually wants NPV/IRR/MSP too).

## 2. Quick start

```bash
# From the folder containing all the files listed in §3:
python -m http.server 8000
# then open http://localhost:8000/equipment_toolkit.html
```

Opening any of the four `.html` files directly as `file://...` will fail silently or show a
"could not load equipment_library.json" message — this is expected; browsers block local
`fetch()` calls under `file://` for security reasons.

## 2b. Saving and resuming an analysis

**Save session** and **Load session** sit in the wrapper's top bar. Save writes one
`viff_session.json` carrying:

- the **original SFF file**, stored once at the top level — so the session file is self-contained
  and the SFF does not have to be kept alongside it;
- the **SFF CapEx Matcher's** unit decisions, in-progress UI state, alias map and cursor position;
- the **SFF OpEx Matcher's** stream decisions and prices, plus the whole utility slate — amounts,
  units, system capacities, hand-matches, and each row's SFF-vs-default price-source choice;
- the **CapEx Estimator's** rows (including hand-edited correlations, bootstrapped entries and
  SFF-cost overrides), the CEPCI, the scale factor and the SFF-escalation choice;
- **Plant Economics'** convention, every input, the factor and census override maps, and the
  consumables table.

Load restores all four tabs and the bridges between them re-fire, so the capital total reaches
Plant Economics over the live `costs-update` path rather than being restored as a stale copy.

**What it deliberately does not store:** anything one tab pushes to another (`costs`,
`streamsBreakdown`). Restoring a stale copy of a live push is how two tabs end up disagreeing.

**Not the same thing as `unit_alias_map.json`.** The alias map is keyed by `unit_type` and exists
to carry confirmed matches *between different flowsheets*; a session is keyed to one flowsheet and
exists to resume it. The session file contains the alias map, not the other way round.

An SFF file loaded by mistake through **Load session** is refused by its `kind` field, with a
message pointing at the right control, and nothing already loaded is touched.

## 2ba. Autosave and the close guard — Round 23

Until Round 23 the Save button was the **only** persistence in the toolkit. There is now also a
snapshot kept in this browser, and a prompt if you close the tab with work that is not in a file.

**What is saved, and when.** One slot in `localStorage`, rewritten four seconds after your last
change. The payload is exactly what **Save session** writes (§2b) — the same builder — so a
restored snapshot and a restored file are the same thing.

**What the note in the top bar means.**

| It says | It means |
|---|---|
| *(nothing)* | there is nothing worth keeping yet |
| `Unsaved changes` | you have just changed something; the snapshot follows in a moment |
| `Autosaved 14:32 · not downloaded` | it is safe against a closed tab, but it is **only in this browser** |
| `Autosaved 14:32` | that, and everything in it is also in a file you downloaded |
| `Autosave unavailable — download the session to keep it` | this browser refused local storage. **Nothing is being kept.** Use Save session |
| `Autosave full — download the session to keep it` | local storage is full; the last snapshot may be stale |

**It is offered, not applied.** Open the toolkit with a snapshot waiting and a bar appears under
the tabs naming the file it came from, how much was in it and when it was taken. *Restore it* or
*Discard*. It is never restored behind your back, because opening the toolkit is at least as often
the start of something new.

**Closing the tab.** The browser asks to confirm only when there are changes that are **not in a
downloaded file**. An autosave does not count: `localStorage` stays on this machine, so a snapshot
is no help to someone closing the tab to carry on somewhere else. An untouched toolkit never asks.

**It is crash recovery, not version control.** One slot, no history. If you want versions — or
anything you can move, share, archive or send to a colleague — that is **Save session**, and it
always will be.

## 2c. Undo and redo — Round 19

**↶ ↷** sit beside Save/Load in the wrapper's top bar, with the last action named in the tooltip
("Undo: Deleted T-101 and its 5 rows"). One history, **25 steps**, shared by every tab — undoing a
deletion made in the CapEx Estimator while you are looking at Plant Economics brings the rows back
*and* brings that tab forward, so you see what changed.

The mechanism is the session save/restore pair (§2b), not a second serialisation of anything:
before a destructive or bulk action, the tab snapshots itself with the same `collect-session`
state it would write to a file, and an undo hands it back through the same `restore-session` path.
That pair is the only description of a tab's state already known to round-trip, and it was already
covered by `test_session.js`. Nothing new had to be trusted.

**What is checkpointed:** deleting a row, deleting or dissolving a module, applying a template,
clearing a review, skipping or confirming a unit, removing a hand-entered stream, bulk imports —
the actions where a mis-click costs work. **What is not:** typing over a number. The field's own
history covers that, and a snapshot per keystroke would fill all 25 slots with one edit.

Loading a session clears the history: it is a different review, and offering to undo into the
previous one would be offering to mix two.

**Limits worth knowing.** The keyboard shortcut (Ctrl/Cmd+Z) only fires when focus is on the
wrapper's own chrome — an iframe's `keydown` does not bubble to the parent document — so the
buttons are the reliable route. The history is not saved in `viff_session.json`, so a reloaded
review starts with an empty one. `selector.html` and `appendix.html` take no part; neither holds
state a reviewer can lose.

> **If you add a checkpoint to a page, check it is top-level.** A `function` declaration written
> inside an arrow function — say inside `window.addEventListener('message', ev => { ... })` — is
> scoped to that arrow function, so `undoCheckpoint` ends up `undefined` everywhere else and every
> checkpoint silently does nothing. `node --check` cannot see this; it is valid JavaScript. Three
> pages were built that way in Round 19. `test_round19_ui.js` §4b now asserts `typeof
> undoCheckpoint === 'function'` on each page that takes part.

## 3. File inventory

| File | What it is | Who/what edits it |
|---|---|---|
| `VIFF_Toolkit.html` | **Generated — never hand-edited.** One self-contained file (~3.3 MB) bundling all eight tool pages and every JSON data file, so it can be double-clicked and used with no web server. Built by `build_viff_bundle.py`, which reads fresh from disk and overwrites it. **It is a copy**: a change to any `.html` or `.json` file does not reach it until the bundler is re-run, and it carries no version stamp, so a stale bundle looks like an edit that silently failed. Rebuild at the end of any round that touches a source file | Every time a page or data file changes |
| `build_viff_bundle.py` | Builds the above. Inlines the shared JS, embeds each page's JSON dependencies (listed in `PAGE_DATA_DEPS` — keep it in sync when a page starts fetching a new file) behind a `fetch` shim, base64-encodes each page into a `<script>` block and swaps the wrapper's `iframe.src` for `srcdoc` | When a page gains a new data dependency |
| `toolkit_charts.js` | Shared chart builders for Plant Economics, the Estimator and (since Round 12) the LCA tab: `pieData`/`pieChart`, `scaleChart`, `fitPowerLaw`, `logSpace`, and the Round 12 composition forms `stackedBars` (negatives below the baseline) and `rankedBars`. Holds two palettes: `PIE_COLORS` for equipment classes, and `CAT_COLORS` — validated against colour-vision criteria — for composition charts. Read the Round 12 handover before changing either | When a page needs a chart form that does not exist yet |
| `equipment_toolkit.html` | Thin wrapper (iframes the eight tools, relays postMessage between them). **Round 6** reordered the tabs to follow the workflow and renamed the two matchers; **Round 6e** added Save/Load session; **Round 7** inserted Life-Cycle Impacts; **Round 11** folded the SFF CapEx Matcher and the CapEx Estimator under one **CapEx** tab with an inner switcher. **Round 19** added the Home tab (now the landing view), `location.hash` routing, and the shared undo/redo history — seven top-level tabs now, eight iframes. The sub-tab buttons keep the ids `tabSFF` and `tabEstimator`, which is what every relay and several suites address them by | Rarely — only if the tab structure itself changes |
| `home.html` | **Round 19.** The landing tab: what the toolkit is, the seven review steps in order with what each produces and a button that goes there, and an SFF / Project PISCES explainer. Stateless — no data file, no session slice, one outbound message type (`switch-view`) that the wrapper already relayed. See §5.0 | When the workflow itself changes |
| `viff_session.json` | *(you generate this via **Save session**)* The whole analysis in one file: the original SFF plus every decision in every tab. Load it to resume exactly where you left off — the SFF does **not** need to be uploaded separately. See §2b | Written and read by the toolkit; not hand-edited |
| `selector.html` | Guided equipment-selection tool | Rarely — mostly stable UI/logic |
| `estimator.html` | CAPEX calculator | Occasionally — e.g. if the SFF bridge or cost engine changes |
| `sff_matcher.html` | SFF upload + fuzzy match + human review (units/CAPEX side) | Most actively developed piece right now |
| `module_templates.json` | 24 complex-unit composition templates for the CapEx Estimator's module feature (Round 16): what submodules a distillation column, compressor train, drying line etc. are normally costed as, as **empty** slots. Carries roles, suggested library ids and what each size has to be derived from — and deliberately **no** sizes, costs, coefficients or factors. See §5.3a |
| `streams_matcher.html` | SFF upload + two-mode OpEx review: **Boundary Streams** (feed/product/waste pricing) and **Utility Duty & Annual OpEx** (per-unit steam/cooling/power duty pricing, Maintenance/Labor %-of-CapEx, annualization) | New/actively developed — see §5.4 and §5.5. Same review-and-confirm philosophy as the SFF CapEx Matcher |
| `plant_economics.html` | **New (2026-09-10).** Fifth tab: the layer above the bare-module sum and below the variable-cost total — FCI/TCI cascade, Turton operating-labour headcount model, cost of manufacture, DCF/NPV/IRR/MSP, an SFF-reported-totals cross-check, and the `tea_report.json` harmonised export. Computes all three conventions at once. See §5.6 | Actively developed |
| `costing_basis.json` | **New (2026-09-10).** Versioned, citable factor sets for Turton / Seider / Towler–Sinnott, plus labour-model coefficients, pressure-factor constants, MACRS schedules and a manually-curated CEPCI table. Every factor carries its published range | **Human-edited.** Edit this, never the HTML, to retune assumptions or add a house convention |
| `add_library_columns.py` | **New (2026-09-10).** One-time migration that added the Phase 0 costing-schema columns to the xlsx. Already run | Archive it |
| `equipment_library.json` | 196-entry canonical cost/capacity library, fetched by the equipment-side tools | **Generated** — never hand-edit. Regenerate from the xlsx (see below) |
| `equipment_library.xlsx` | The actual master copy of the equipment library (`Library` + `Legend` sheets) | **Human-edited.** Add rows here, with a unique lowercase-hyphenated `id` |
| `build_equipment_library_json.py` | Regenerates `equipment_library.json` from the xlsx | Run this after every xlsx edit |
| `decision_trees.json` | Guided Q&A trees, category list, family/cost-chart groupings, all pre-merged | Static-ish; only touch if the decision tree structure itself needs to change |
| `toolkit_format.js` | **Round 5.** Shared typography: `TK.sub()` for subscripted symbols, `TK.eq()` for typeset equations, plus a `MutationObserver` that reapplies both after every redraw. Loaded by all six pages | Edit only to add a symbol to its `SYMBOLS` whitelist |
| `chemical_prices.json` | **Round 6d.** Prices for the chemicals, feedstocks, products and disposal routes that cross a plant boundary — 19 sourced entries, matched by CAS first. Tier 4 of the boundary-stream price chain, and the only tier that can price a stream the SFF left alone | **Human-edited.** Every entry needs a source, a URL/DOI, a basis year and a `$/kg`-normalisable unit, or it does not belong in the file |
| `toolkit_charts.js` | **Round 6.** Shared, DOM-free SVG builders for the equipment cost-breakdown pie and the total-cost-vs-scale plot, plus the fitting helpers and log-spacing. **Round 20** added two more fitted forms beside the power law — `fitQuadratic` and a continuous `fitPiecewiseLinear` — behind one `fitCurve(xs, ys, {form, pieces})` dispatcher, plus `fitSummaryHtml` so the three panels that print the fitted equation print the same wording. Loaded by the CapEx Estimator and Plant Economics | Edit here, never in a page — the whole point is that there is one implementation |
| `unit_alias_map.json` | *(you generate this via SFF CapEx Matcher)* Confirmed `unit_type → library id` mappings, carried forward across SFF files | Exported/re-uploaded by the reviewer each session |
| `stream_pricing.json` / `.csv` | *(you generate this via SFF OpEx Matcher, Boundary Streams mode)* Per-stream category, price, source, and cash flow for one SFF file | Exported by the reviewer each session; nothing is re-imported across sessions yet (see §7) |
| `utility_opex.json` | *(you generate this via SFF OpEx Matcher, Utility Duty & Annual OpEx mode)* Every duty row (unit, matched utility, match type, $/hr) plus the Annual OpEx settings and summary (TIC, maintenance %, labor %, annual hours, total annual OpEx) | Exported by the reviewer each session; no alias-map equivalent yet (see §7) |
| `SFF_streams_scrapper.html` | **Superseded** by `streams_matcher.html`. Kept only for reference — it assumed a schema (`stream.roles`, a flat `stream.price` number) that doesn't match real SFF files (see §6). Safe to delete once you've confirmed the new tool covers your needs | Don't edit; retire it |
| `test_*.js` / `test_workbook_recalc.py` | Regression suites, sitting beside the HTML files (**not** in a `tests/` subfolder — that layout is gone). Ten run in plain Node, the rest need Playwright, one also needs LibreOffice. **1,906 checks as of Round 23** (744 Node, 1,080 browser, 82 workbook). | See `README.md` for what each one covers and how to run it. Add one whenever a feature is added |

**Never duplicate the equipment library data.** The entire point of this architecture is that the
equipment-side tools read the *same* `equipment_library.json` — if you ever find yourself pasting
equipment data directly into one of the HTML files again, something has gone wrong.

## 4. Data flow

**CAPEX / equipment side** (unchanged by this update):

```
                         ┌─────────────────┐
   SFF file  ──upload──▶ │  SFF CapEx Matcher    │
                         │  (fuzzy match + │──confirmed/bootstrapped──┐
   unit_alias_map.json──▶│   human review) │      items               │
   (optional, from a     └────────┬────────┘                          ▼
    previous session)             │                          ┌─────────────────┐
                                  ├─ unit_alias_map.json  ◀──▶│  CapEx Estimator│◀── picks from
                                  ├─ verification_record.json │  (running list, │    Equipment
                                  └─ new_library_entries.json │   charts, export)   Selector
                                         │                    └─────────────────┘
                                         ▼
                              (human merges into)
                              equipment_library.xlsx
                                         │
                              build_equipment_library_json.py
                                         ▼
                              equipment_library.json  ◀── fetched by both equipment-side tools
```

**Streams / OpEx side:**

```
                         ┌───────────────────────────┐
   SFF file ──upload──▶  │   SFF OpEx Matcher  │
                         │                           │
                         │  Boundary Streams mode  ──┼──confirmed/skipped──▶ stream_pricing.json
                         │  (3-tier price suggestion, │                      and .csv
                         │   human review)            │
                         │                           │
                         │  Utility Duty & Annual  ──┼──────────────────────▶ utility_opex.json
                         │  OpEx mode (§5.5): duty     │
                         │  matching, Maintenance/     │
                         │  Labor %, annualization    │
                         └─────────────┬─────────────┘
                                       │ request-tic / tic-update
                                       ▼
                              ┌─────────────────┐
                              │ CapEx Estimator │  (same Estimator as the CAPEX side above)
                              │ Total Installed │
                              │ Cost, live      │
                              └─────────────────┘
```

**The bridge between the two OpEx tabs:** the *same raw SFF file* feeds both the SFF CapEx Matcher and
the SFF OpEx Matcher without a second upload. Whichever tool's "Start review" button is clicked
first broadcasts the raw parsed JSON via `postMessage`; `equipment_toolkit.html`'s wrapper relays
that broadcast to whichever of the two SFF-consuming tabs didn't send it. Each tool still parses
that raw file independently for its own purpose (units → CAPEX, streams/duty → OpEx) — there's no
shared in-memory data structure, just a shared raw-file hand-off, so the two tools' parsing logic
stays fully decoupled. A tool only accepts an incoming broadcast if it hasn't already started its
own review locally, so switching tabs never clobbers work in progress.

**The bridge to the Estimator (Total Installed Cost sync):** the Annual OpEx panel's Maintenance/
Labor "% of TIC" fields need a real dollar figure to multiply against, so on `beginReview()` the
SFF OpEx Matcher posts a `request-tic` message; the wrapper relays it to the Estimator tab,
which responds with a `tic-update` message carrying its current running total (relayed back the
same way). The Estimator also pushes `tic-update` on every edit and once on load, so a tab opened
in either order still ends up in sync — this is the exact same request/push pattern already used
for the CEPCI sync between the Estimator and SFF CapEx Matcher, just carrying a different number to a
different tab. If the Estimator tab is never opened (or the tool is opened standalone), the TIC
field in the Annual OpEx panel is just a plain editable number instead.

## 5. Feature reference

### 5.0 Home (`home.html`) — Round 19

The tab the toolkit opens on. Three things and nothing else:

1. **What this is** — a paragraph a new collaborator can read in thirty seconds.
2. **The review, in order** — the seven tabs as seven steps, each with a button that switches the
   wrapper to it. **Round 21** put CapEx ahead of OpEx so the list matches the tab bar above it;
   before that the page and the navigation gave different answers to "where do I start". The list
   also now says what was always true and undocumented: **the SFF file goes in once, on either
   matcher.** Both parse it and each broadcasts it to the other, so the order is a recommendation
   rather than a constraint — the two sides are independent until Plant Economics adds them up.
   (The OpEx tab is still the only stream parser, so the plant's own capacity is derived there.)
   The five *working* steps each state what they **produce** on their own line
   ("produces → Σ C_BM, the purchased-cost sum, and the equipment-class census"); the last two
   (Equipment Selector, Parameters & Basis) are reference tabs and carry no such line. The
   "produces" line is the point: the order is not arbitrary, and each step is there because the
   next one needs its output.
3. **What an SFF file is** — the Standard Flowsheet Format the whole toolkit reads, in Project
   PISCES' own words (https://projectpisces.org/): an open, non-proprietary JSON standard where
   unit operations are nodes and streams are edges, extracted from papers and simulators by
   multimodal AI **with a human in the loop**. The four top-level parts this toolkit reads are
   called out — `metadata`, `units`, `streams`, `chemicals` — each named with the tab that
   consumes it. A closing note states the accuracy class out loud (**planning-level, ±30–50%**,
   not vendor quotes) and the two rules the whole codebase is held to: every number carries a
   source, and where the data does not say, the toolkit says so.

It holds no state, fetches no JSON, has no session slice, and posts exactly one message type —
`switch-view`, which the wrapper already relayed for the Estimator. It therefore costs no new wire
protocol and cannot drift from what the other tabs compute, because it computes nothing.

It also does **not** know where you are in a review. It could ("12 line items, 3 unpriced streams,
next is the reboiler slot"), which would make it a dashboard rather than documentation — a bigger
idea, and deliberately not Round 19's.

### 5.1 Equipment Selector (`selector.html`)

- **Guided view**: category → Q&A decision tree → result, mirroring the structure of
  `equipment_cap_library.xlsx`'s original 29 category sheets.
- **Decision tree map**: a visual map of the current category's whole tree, click any node to jump.
- **Single-cost-chart**: log-log cost-vs-size chart for one equipment entry.
- **Family comparison chart**: overlays several related entries (e.g. all tray types) on one chart
  so you can see relative cost-effectiveness across a size range.
- **Two-stage selection** (Round 4): stage 1 picks the equipment *type*, stage 2 picks the
  *material of construction* for it. A stage rail shows where you are and the pair is sent to the
  Estimator in one message, so the row lands with its material already selected rather than on the
  library default.
- **Material sheet (SH.11) is directly reachable** (Round 6). Round 4 had removed it from the nav
  to stop people treating "pick a material" as an alternative to "pick a pump" — right about the
  order, wrong about access: it left the sheet reachable *only* by walking a full equipment tree to
  a result, so you could not read the material tree or look up an `F_M` on its own. It is back, in
  its own **Reference** group at the foot of the nav rather than among the equipment sheets, with a
  note saying it is normally stage 2. Opening it that way is explicitly a browse: the panel says no
  equipment has been chosen, and a material result offers *"pick an equipment type and this material
  will be sent with it"* instead of a Send button that would silently do nothing. The choice is
  remembered, so finishing an equipment tree afterwards offers **"Send with «material»"** and the
  pair goes across exactly as the two-stage route would have sent it.
- **The stage rail is a tab pair** (Round 10). *Equipment* and *Material*, side by side at the top
  of the nav, both clickable, one selected at a time. Before this the rail looked like tabs and was
  two inert `<div>`s, so the only route to SH.11 was the Reference entry below 28 equipment sheets —
  a scroll away, behind a control that appeared to be the right one and did nothing.
  Being clickable does not make material an alternative to picking a pump: entering the Material tab
  while stage 1 sits on an equipment **result** carries that result in as stage 2, exactly as the
  *Next: choose material* button does; entering it from anywhere else is a browse, and the panel says
  so. Stage 2 is dimmed when nothing is picked yet, never disabled — a sheet you may read is a sheet
  you may click.
- **Send to Estimator**: posts the chosen entry (by name + optional size) to whichever tab is the
  Estimator; the wrapper switches to the Estimator tab so the user sees it land.
- **Opened from the CapEx matcher**: the Selector accepts one message,
  `{source:'wrapper', type:'open-material-guide'}`, and opens SH.11 on it. `selector.html#material`
  does the same on load, for when the page is opened on its own rather than inside the toolkit.

### 5.2 CapEx Estimator (`estimator.html`)

- **Library tab**: pick category → equipment → material → size → quantity → add to the running
  list. Live per-row cost breakdown (base cost, CEPCI-adjusted cost, installed-cost factor,
  total installed).
- **Custom Equipment tab**: define an ad-hoc correlation by hand (for something not in the
  library at all) — this predates the SFF CapEx Matcher and is still useful for one-off cases.
- **Converter tab**: general unit conversion (length, area, volume, mass, flow rates, power,
  pressure, energy, temperature) — the same engine is reused inside the SFF CapEx Matcher's
  harmonization panel.
- **CEPCI year picker** (Round 6): the *Current CEPCI Index* cell carries both a free-typed box
  and a year dropdown. The dropdown is built from `costing_basis.json`'s `index_by_year` — the
  same curated table the Appendix prints — so it cannot drift from the published values the rest
  of the toolkit cites. Picking a year writes that index into the box; typing a value the table
  does not hold drops the picker to *Custom*, so the UI never claims a year whose index is not in
  force. Years past `unverified_after` are labelled *(provisional)* and turn the caption amber,
  because the file says in so many words that those are placeholders awaiting a subscription
  value. Changing the year rebroadcasts `cepci-update`, so utility prices in the SFF OpEx Matcher
  move with it.
- **SFF cost-basis year** (Round 6b): when the loaded SFF states `metadata.TEA_year` *and* at
  least one row is an SFF-reported cost, a bar appears above the table. SFF-reported costs are
  passed through verbatim while every library row beside them is escalated to the Current CEPCI
  Index — so without this the total silently mixes two dollar years. The bar states the year, the
  CEPCI for it, how many rows are affected and what they total, and offers a checkbox to escalate
  them from that year. **Off by default**, because switching it on changes a dollar figure and a
  file arriving must never do that by itself. The choice is written into the CSV, the report and
  the `costs-update` provenance. A year with no entry in `costing_basis.json` disables the control
  and says why rather than falling back to 1× silently. (In the MSW example this is worth 1.477× on
  $22.2M of reported costs.)
- **Breakdown tab**: pie chart of total installed cost, grouped by line item or category.
- **Scale tab**: line chart of how the running total would change if every quantity scaled by a
  factor (for quick sensitivity checks). Note the clamp: `computeCost()` never evaluates a
  correlation below its `size_min`, so a row sized under its own minimum contributes a *flat*
  segment to this curve. That is deliberate — the tool does not extrapolate a correlation
  downward — but it means a sweep that comes back flat usually means an undersized row, not a
  broken chart.
- **Both charts are also drawn on the Plant Economics tab** (Round 6), from the same builders in
  `toolkit_charts.js`. The sweep itself is still computed *here*, because this is the only page
  that holds the correlations; Plant Economics asks for it over `request-scale-sweep` and gets
  back the (scale, installed cost, Σ C_BM, Σ C_BM-at-base) curves to run its own cascade over.
- **Modules** (Round 16): line items can be grouped into named complex units — a distillation
  column read as its shell, trays, condenser, reboiler, reflux drum and pumps, with a subtotal —
  and built from a template as a checklist of empty slots. A module owns no cost of its own. Full
  reference in §5.3a.
- **Export**: CSV of the line-item table, and a self-contained HTML report snapshot. Both are
  grouped by module, and both state any submodule slots left uncosted.
- **Accepts items from two senders**: the Equipment Selector (`{name, size}`, looked up in the
  existing library) and the SFF CapEx Matcher (`{name, size}` for a real match, or `{entry, size}` for a
  bootstrapped/manual entry that isn't in `equipment_library.json` yet — that gets added as a
  one-off row tagged `custom: true`).

### 5.2b The CapEx tab — one tab, two halves (Round 11)

Matching a flowsheet's units against the cost library and costing them are one job in two moves, so
they share one top-level tab. A second strip under the main one carries the two halves — **SFF CapEx
Matcher** (§5.3) and **CapEx Estimator** (§5.2) — and is drawn only while CapEx is the active tab.

- **Both pages are unchanged and both stay loaded.** Switching halves does not rebuild, reload or
  reset anything: a review in progress in the matcher is exactly where you left it after a trip to
  the estimator and back. Nothing about either tool's behaviour changed in this round.
- **The tab remembers which half you were last on**, so coming back from Plant Economics half way
  through costing a list does not look like the list was thrown away.
- **The count on the Estimator sub-tab** is how many rows are in the estimator right now. Confirming
  a unit in the matcher deliberately does *not* switch the view — you work through units in a row —
  so the count is the evidence that the confirmation landed. It is the estimator's own figure, read
  off the `costs-update` message it already broadcasts on every edit, so it falls again when a row is
  deleted there; it is not a tally of messages the wrapper has relayed.
- **The tools keep their names** on the sub-tabs. Grouping them changed the navigation, not the
  tools — and the rest of this manual, plus about fifty notes across the other pages, sends readers
  to one of them by name.

> **Round 19 renamed three labels.** The top-level tabs read **CapEx (Equipment)** and **OpEx
> (Stream/Utility)**, so each says which cost it covers *and* what it is keyed off — the pair was
> "CapEx" and "OpEx", which told you the first and not the second. Inside the OpEx tab, the view
> that was **Cost summary** is now **Stream summary**, because that is what it summarises; the
> cost summary a reader would expect under that name lives on Plant Economics (§5.6b). These are
> labels only. Every `data-view` key, every element id and every `postMessage` `source` string is
> unchanged — `summary` is still `summary` — which is the rule in `TODO.md` §0.5: rename the
> label, leave the key.

### 5.3a Modules and complex-unit templates (`estimator.html`, `module_templates.json`) — Round 16

> **Round 20 — a module carries its flowsheet unit id.** A module created from an SFF unit is
> named `Shredding (U-1)`: the display name and the id, because either alone is ambiguous. A
> SuperPro export calls half its units `Generic`, so a module named only for the display name is
> three modules called Generic; a module named only for the id tells you nothing about what it is.
> When the display name *is* the id, or there is none, the brackets are dropped. The small tag
> beside the name is the **unit id**, so renaming a module to something meaningful does not lose
> the only thing that traces it back to the file.

A trayed distillation column is seven library rows: a shell, a set of trays, a condenser, a
reboiler, a reflux drum and two pumps. Until Round 16 the Estimator had no idea they were one
thing, so "what does the column cost" had no answer and "was the reboiler ever costed" had no
answer either. A **module** is a named group over line items; a **template** is a checklist of the
submodules a complex unit is normally costed as.

**A module owns no cost.** This is the property everything else rests on: a module has no
correlation, no factor and no size of its own, and its subtotal is the plain sum of its rows'
bare-module costs. Group a loose row into a module, or dissolve a whole module again, and Σ C_BM,
Σ purchased and the base-condition sum are identical to the cent. `test_round16_ui.js` §4 asserts
it in both directions, and it is the reason this was safe to add to a costing page at all.

**Working with modules**

- Every row carries a small **module picker** in its equipment cell: *not in a module*, any
  existing module, or *+ new module…*. That is how a list built before Round 16 gets grouped, and
  how a row is moved from one module to another.
- A module header shows an **editable name** (which travels into the CSV, the HTML report and Plant
  Economics), the template or SFF unit it came from, its submodule count, its **subtotal**, and how
  many of its submodules are not costed yet.
- **⤴ ungroup** dissolves the module and leaves the line items in the list; **✕** deletes the
  module *and* its rows. Two buttons on purpose: the first changes nothing about the total, the
  second changes it, and confusing them costs a reviewer their work.
- **+ sub** adds another empty slot to a module. Rows are re-sorted so a module's members sit
  together, which also keeps the `#` column, the CSV and the broadcast row detail in one order.

**Templates (`module_templates.json`)**

24 complex units: trayed and packed distillation, absorber/scrubber, stripper/regenerator,
liquid–liquid extraction, stirred-tank reactor, fired heater, steam plant, compressor train, pump
set, multi-effect evaporator, crystallisation train, spray drying, rotary drying, size-reduction
circuit, filtration, membrane skid, vacuum system, gas cleaning, cooling water, wastewater,
storage & transfer, flash drum and a heat-recovery set.

Each slot names a **role**, suggests library ids that fill it, hints the class and categories to
widen the picker to, and says **what the size figure has to be computed from** — in words. Nothing
is pre-selected: a template suggests, it does not choose.

**No template carries a size, a cost, a coefficient or a factor.** A template that carried a number
would be a second cost model, which is exactly what this toolkit is built not to have. What a
template does carry is the reference its composition came from (Seider §16, Sinnott & Towler
§17–19, Turton App. A) and, where those three disagree about whether an item is inside the module or
beside it, an optional marking plus a note saying why. Several notes are **warnings**, and are worth
reading before ticking an optional slot:

- the jacketed-agitated reactor rows already include their agitator **and** their jacket;
- Seider's `compressor, centrifugal, electric motor (preferred)` and its siblings already include
  their driver, while a bare Sinnott pump correlation already includes its driver too;
- `cooling tower w/ pumps, field assembled` already includes its pumps.

Ticking the matching optional slot beside any of those pays for the same equipment twice.

**Unresolved slots — the part to understand before trusting a total**

A template creates **empty** rows: no class picked, no size entered. Such a row is *unresolved*, and
an unresolved row:

- contributes **nothing** to Σ C_BM, Σ purchased or the base-condition sum;
- does not appear in the `equip_class` census, so Turton's operating-labour model is not staffed for
  a processing step that does not exist yet;
- does not appear in the CSV table, the HTML report table, or the per-row detail broadcast
  downstream;
- **is counted, out loud, everywhere the total is.** The totals bar reads `4 line items · 1 module ·
  2 NOT COSTED`; the module header says `2 not costed yet`; the CSV gets a NOT-COSTED block naming
  each one; the HTML report gets an amber box listing them; and `costs-update` now carries
  `unresolved`, so Plant Economics states on its capital panel that every figure below it — FCI,
  TCI, COM, the minimum selling price — is short by whatever those slots would cost.

The alternative was to pre-fill a plausible class and let the size floor in `computeCost()` cost it
at `size_min`. That is a guessed number in a real number's clothes. A null beats a guess.

A slot becomes a normal line item the moment it has **both** a class and a size, and from then on it
behaves exactly like any other row — same cells, same editor, same arithmetic. That last point is
checked rather than assumed: `test_round16_ui.js` §3 adds the same pump through the pre-module
library panel and as a resolved template slot and requires the two to agree to the cent, and both to
agree with the published correlation evaluated by hand.

**Filling a slot (Round 17)**

A slot has two states and both are visible. Before a class is picked it shows the template's own
guidance — what the size has to be derived from, and any warning attached to that role. The moment
a class is picked the dropdown **shows it**, and the guidance is replaced by the entry's own facts:
its name, category and source, the unit the size has to be in, the tested size range and the base
material. The material picker, the size range and the F<sub>P</sub> cell go live at the same point,
so a material or a design pressure can be set before the size is known.

It still says **enter a size**, still takes no line number and still costs nothing. Type a size and
the row becomes a costed line item **in place**, without the table being rebuilt — which matters,
because rebuilding it destroys the box being typed into: before Round 17 a size typed digit by digit
arrived as its first digit, so 9000 became 9 with a plausible cost beside it. Clear the size and the
row goes back to being an open slot, with the size stored as null rather than zero, because "not
entered" and "zero" are different claims.

**From an SFF file, this happens by itself**

The SFF CapEx Matcher has supported confirming one flowsheet unit as several components since
Round 6 (*confirm & add another component to this unit*), but it used to post them with no reference
to the unit they came from — which is why a column reviewed from a file still landed as four
unrelated rows. Every posted item now carries `sff_unit_id`, `sff_unit_name` and `sff_unit_type`,
and the Estimator creates one module per unit on first sight and reuses it for every later component
of that unit. Matching is on the **id**, so renaming the module in the Estimator does not make the
next component start a second group.

Those rows are labelled with a `role` and are **not** slots: they arrive with a class and a size
already chosen, and several library rows have `size_min = 0`, so treating them as slots would have
let a component sized at zero drop silently out of the plant total.

**The matcher suggests the module, not only the item (Round 17)**

Above its per-item matches, the SFF CapEx Matcher now carries an **Is this a complex unit?** card. It
ranks `module_templates.json` against the unit's own words — its PISCES type name, with the display
name corroborating — and shows why each suggestion matched. This is what the matcher was missing: on
a real file its best answer to "this is a distillation column" was "this is a tray, score 0.13".

Scoring is deliberately narrow. Only **single-word aliases** count as tokens; a multi-word alias is
matched as a phrase with separators ignored, which is what finds "Waste Water Treatment" under
`wastewater`. The template's own name is not scored at all — "Flash / knockout drum with pump-out"
used to make that template a candidate for every pump on the flowsheet. A unit that is one library
row — a heat exchanger, a mixer, a centrifuge — suggests no module, and that is checked.

Applying a template opens its checklist, ticked as core/optional exactly as in the Estimator's own
panel, and creates the module with the ticked submodules as empty slots. Applying it twice does not
double it.

Each confirm panel then carries a **fills which submodule** picker listing the slots no component has
claimed yet. The item travels with a `slot_role` and the Estimator fills that open slot in place
rather than adding a row beside it, so a column reviewed component by component ends up as its
checklist ticked off rather than a checklist plus five loose rows. The preselection uses two rules —
the template lists this entry id for that role, or exactly one open slot wants this entry's
`equip_class` — and **refuses to guess** otherwise: with a condenser and a reboiler both open,
picking one for you would label a million-dollar exchanger wrongly, and you are right there.
"— none of these —" is always offered and adds the component to the module as its own line item. A
role that is already costed is never silently overwritten.

The unit rail shows `module 1/5` rather than `confirmed`, because three of five submodules confirmed
is a unit still missing two costs.

**Working one submodule (Round 18)**

Each line of an applied module's checklist is something you act on, because a submodule is a piece of
equipment that happens to have a named place in a bigger unit:

- **Match this submodule** on an open one scopes everything below it — the candidate list, the size
  box, the material picker, the confirm button. A banner names the submodule, repeats the template's
  guidance for that role where the choice is actually made, and offers the way back to matching the
  whole unit.
- **Edit / verify** on a confirmed one reopens exactly what is recorded: the entry it was matched to,
  the size, the material, and for a bootstrapped or manual entry its coefficients. Re-confirming
  replaces the record — keeping both timestamps, so the verification file says it was revised — and
  **updates the Estimator row in place** rather than adding a second one. Change the class as well as
  the size and the row follows.
- **Remove** takes it off the checklist and reopens the slot here. The Estimator row stays costed;
  the note beside it says so, because deleting a row you may have edited there is not something this
  tab does unasked.

The candidate ranking follows the **role**, not the unit's name. The fuzzy score against the unit
survives as a tie-breaker, and the template's own knowledge leads: +2.0 where the template names that
entry for that role, +1.0 for a category the role names, +0.5 for its equip_class. So a reboiler slot
ranks the thermosiphon and U-tube kettle reboilers first, each saying why it is there, and the column
shell ranks pressure vessels — where matching against "Distillation Column" would have offered trays
to both. Nothing is hidden: the tiers re-order the list, and *Search the full library* is unchanged.

An ordinary confirm still fills an **open** slot only. The picker offers no role that already has a
component, and an item aimed at a costed role without the edit flag adds a visible duplicate instead
of overwriting — replacing a figure somebody has already checked is the failure mode being guarded
against, and a duplicate you can see is not.

**Where the grouping shows up**

| Where | What it adds |
|---|---|
| Estimator table | module header, subtotal, indented submodules with role labels, collapse |
| Totals bar | module count and the NOT-COSTED count |
| Cost Breakdown | a third grouping, **By module** — a column reads as one slice, not seven |
| CSV export | `Module`, `Role in Module`, `Module Template` columns, plus a NOT-COSTED block |
| HTML report | equipment list grouped under module headings with subtotals, plus an open-slot box |
| `costs-update` | `module`, `moduleId`, `moduleTemplate`, `role` per row; `unresolved` and `moduleCount` on the message |
| Plant Economics | module and role on each equipment row of the traceable report; the open-slot caveat on the capital panel and the Σ C_BM card |
| Session file | `modules[]` plus `moduleId`, `slot` and `role` per row. A session saved before Round 16 has none of these and loads ungrouped, at the total it had |
| SFF CapEx Matcher | a ranked module card per unit, a slot checklist, a fills-which-submodule picker on every confirm, and `module N/M` on the unit rail (Round 17) |

**What this deliberately does not do:** no module-level cost object or grouped F_BM; one level of
nesting only; no auto-sizing from a template; and no name-similarity heuristic for grouping — rows
group when a reviewer groups them or when an SFF file says two components are one unit, and never
because two names looked alike.

### 5.3 SFF CapEx Matcher (`sff_matcher.html`)

**Upload**: an SFF JSON file, plus optionally a previous `unit_alias_map.json`.

**Dual-mode parsing.** SFF files come in at least two shapes, and the parser handles both,
including a mix within the same unit:
- *Literature-extraction style*: every value wrapped as
  `{value: <number>|{value, units}, source:{snippet, location, ...}, confidence:{level, ...}}`.
- *Simulation-export style*: flat numbers directly on `design_results`, `purchase_costs`,
  `installed_costs.equipment_installed_cost`, and `equipment_details.size`/`size_units`.
- Both `design_input_specs` *and* `design_results` can appear in either shape — e.g. a Steam
  Turbine's `design_results.electricity_production` in one real example file was literature-style
  wrapped, not a bare number. The parser now checks every field the same way rather than assuming
  a shape per section (this was a real bug, fixed and tested — see §6).

**Per-unit review panel** shows:
- **Connected streams** — every stream whose source or sink is this unit, with direction (in/out),
  mass flow, volumetric flow, temperature, and pressure. Shown first, before specs or candidates,
  because it's the context that actually tells you whether a candidate match is plausible in the
  first place — a "Distillation Column" candidate list means very little without first seeing what
  is actually flowing into and out of it.
- Design specs table — value, **unit** (or an explicit "no unit given" flag, never silently
  blank), source snippet/location, confidence level.
- Reported cost table (installed cost; purchase cost shown as secondary context if present).
- **Components already recorded for this unit**, if any (see "Multiple components per unit"
  below) — each with a "Remove" button, in case one was added in error.
- Previously-confirmed alias(es), if this exact `unit_type` string was matched before — see below,
  since one unit type can now remember more than one.
- Fuzzy-matched candidates (ranked, scored against the unit's PISCES type name + display name).

**Multiple components per unit.** One SFF unit sometimes needs more than one library entry to
cost properly — a distillation column, for instance, is really a vessel *plus* trays or packing
*plus* a reboiler and condenser (separate heat exchangers), all reported as a single unit in the
SFF. Every confirm action (matched, bootstrapped, or manual) has two buttons:
- **"...& finish this unit"** — the default, one-click path for the common single-component case:
  commits this component and moves to the next SFF unit, exactly like before this feature existed.
- **"...& add another component to this unit"** — commits this component but resets the panel back
  to the candidate-picker view *for the same unit*, so you can immediately pick/bootstrap/manually
  add the next component (e.g. confirm the vessel, then add another for the reboiler, then again
  for the condenser, then finally "...& finish this unit" on the last one).

Every component is sent to the Estimator as its own row the moment it's confirmed — there's no
batching or "send all at once" step. The sidebar badge shows a count once a unit has more than one
component (`confirmed (3×)`), and the alias map now remembers *all* confirmed library ids per
`unit_type` (not just one) — so the next "Distillation Column" unit in the same or a future SFF
file shows all previously-confirmed components (vessel, reboiler, condenser, ...) as one-click
"previously confirmed mapping" shortcuts, instead of forcing you to re-search for each one. Older
exported alias maps (one object per unit_type, from before this feature) are still read correctly
— they're just treated as a one-entry list.

**Five things you can do with each component:**
1. **Accept an alias** (one click, if a prior session already confirmed a component for this unit
   type — possibly several, if it's a multi-component type).
2. **Accept a fuzzy candidate**.
3. **Search the full library** — free-text + category filter across all entries, independent of
   the fuzzy scores. Needed because fuzzy matching against a generic PISCES type name (e.g.
   "Mixing", "Distillation") often scores every candidate at zero even when a good entry exists —
   this was added specifically because the algorithm can't see what a human immediately recognizes.
4. **Bootstrap** a brand-new entry from the SFF's single reported cost point — default scaling
   exponent n = 0.6 (six-tenths rule), editable, must tick "I've reviewed these values" before
   confirming. Valid size range (min/max) defaults to a rough ±50% bracket around the single point
   and is always editable — a single data point can't fit a real range, so this is clearly a
   guess, not a fitted bound.
5. **Add manually** — blank entry shell for when there's no cost to bootstrap from either; flagged
   as needing a real correlation before use.
   Or **Skip** (renamed "Done with this unit" once at least one component is already recorded, so
   it's clear this won't discard anything already added) and come back later.

**Material of construction** — a dropdown in the matched panel, parsed from the same "Pricing
Materials: Name (factor); ..." text the Estimator embeds in each entry's `note` field, so both
tools always agree on the same options. Defaults to whichever option matches the unit's own
reported `material_of_construction` (exact or partial match), falling back to the entry's base
material. The selected material's factor feeds the cost comparison below, and the selection is
passed through to the Estimator on confirm — so a stainless-steel unit lands in the Estimator
already set to stainless, not silently defaulted to base material.

**Size to evaluate at** — always shown (even for units with no SFF-reported cost to compare
against), because this is what actually gets sent to the Estimator on confirm. Auto-fills with a
sensible default the moment a match is picked (from the same spec the converter widget guesses
from), and can be typed over directly or set via the converter below — either way, whatever's in
this field when you click Confirm is what the Estimator receives, not a leftover blank that
silently defaults to the entry's minimum size (a real bug that shipped before this was fixed; see
§6 if you're wondering why this sentence sounds defensive about something so basic).

**Cost harmonization** (comparison + resolution choice only shown once a match is picked, if the
SFF also reports a cost — the size field and unit converter above are always available):
- **Unit converter widget** — the library entry's own size basis is always the master unit (this
  is what actually gets sent downstream), so it anchors the category, and defaults to whichever
  SFF spec is dimensionally compatible with it (not just the first spec listed). Fully overridable
  manually, and a button drops the converted number straight into the size field. Falls back to
  the spec's own detected unit with a visible warning when the entry's size basis can't be
  auto-detected at all — either because its wording isn't recognized, or because it's one of the
  8 composite "S = (...)*(...)^n" formulas (e.g. a pump's "S = (Flow rate, gal/min)*(Pump head,
  ft)^0.5") that aren't a single physical unit a converter can bridge to in the first place.
  Resets automatically if you switch to a different candidate mid-review.
- Predicted (library formula, CEPCI/**selected-material**/installed-cost-factor-adjusted) vs.
  reported cost, with a % delta. The comparison card explicitly lists which material factor and
  installed-cost factor were applied, so this is auditable rather than a black box.
- Resolution choice: **trust the library correlation (default)** and log the SFF value as
  reference only, or use the SFF-reported cost for this instance. Both choices are recorded in the
  verification record either way — but see the known limitation in §7 about what "use SFF cost"
  actually does downstream.

**Sends to the Estimator** on every confirm (matched, bootstrapped, or manual-with-real-numbers) —
a toast confirms it in the SFF CapEx Matcher's own tab; the wrapper does **not** switch tabs away from
the Matcher for these (unlike the Selector's one-off send), since reviewing an SFF file is a
batch, many-units-in-a-row workflow.

**Exports** (all downloadable any time):
- `unit_alias_map.json` — feed this back in next session so previously-confirmed mappings don't
  need re-confirming from scratch. Each `unit_type` now maps to an **array** of `{library_id,
  confirmed_by, date}` entries (one per remembered component), not a single object — a
  distillation column's vessel, reboiler, and condenser can all be remembered under the same
  `unit_type`. A map exported before multi-component support (a single object per unit_type) is
  still read correctly; it's just treated as a one-entry list.
- `verification_record.json` — full audit trail: per-unit `components[]` array (one entry per
  confirmed component — action taken, chosen library id, material, size, cost comparison numbers
  and which resolution was picked, timestamp each), plus a `skipped` flag if the whole unit was
  skipped with nothing recorded.
- `new_library_entries.json` — bootstrapped/manual entries in the exact schema
  `equipment_library.xlsx` expects, ready to be pasted in as new rows (then re-run
  `build_equipment_library_json.py`). Flattened across every component of every unit, not just
  one per unit, since a unit can now have several bootstrapped/manual components.

**Material of construction, and where the guidance is** (Round 10). The matched panel's material
dropdown lists the options the library entry itself declares, each with the factor it multiplies the
bare cost by — a range of roughly 1x to 3x, which makes it one of the more consequential fields on
the panel. It carries a **Which material? Open the selection guide →** link (as does the bootstrap
panel's free-text material field), which switches the toolkit to the Equipment Selector and opens
SH.11 there; standalone it opens `selector.html#material` in a new tab. The matcher deliberately
holds **no material tree and no factor table of its own** — same rule as the single SFF parser: a
second copy is a copy that drifts. It links to the real sheet instead.

### 5.3b Checking and changing a unit you already confirmed — Round 20

Confirming a unit used to be one-way. The card listed what had been confirmed as a name and a
size, and everything that made it a *decision* — where the size came from, which material, how the
entry was found, and whether the library correlation or the SFF-reported cost was used — lived
only in the export.

Every recorded component now states all of that on its card, and carries a **✎ Edit / verify**
button that reopens the selection panel with those choices still in it: the library entry picked,
the size, the size basis back on the spec it was read from, the material, and the cost comparison
with the resolution radio still selected. The panel says in so many words that it is editing.

**Re-confirming updates the CapEx Estimator row in place.** It does not add a second one. Each
component carries a `component_id`, issued once and kept across every revision, and the Estimator
stamps it on the row it creates — so a revision finds its own row however the list has been
reordered or regrouped since. The component record keeps both timestamps: `first_confirmed_at` and
`revised_at`, because what a reader of the verification file needs to know is that a figure was
revised and when.

Round 18 did this for a submodule bound to a module **slot**, keyed on its role. A role is a
position, so it could not distinguish two components in the same role, or find one in no role at
all. The component id is the identity that makes it work for anything.

**Cancel** leaves the component exactly as it was and clears the draft — an abandoned edit does not
sit half-finished behind the next confirmation.

### 5.4 SFF OpEx Matcher — Boundary Streams (`streams_matcher.html`)

**Upload**: an SFF JSON file — the same file as the SFF CapEx Matcher accepts. If it's already been
uploaded in the SFF CapEx Matcher tab (or gets uploaded there next), this tab picks it up automatically
via the broadcast bridge described in §4 — no second upload needed. Uploading directly in this tab
works too, standalone or embedded.

**Or no file at all (Round 9).** The upload screen's second button, *Start with no file — enter
everything by hand*, begins an empty slate: a process on paper, a literature flowsheet, or a screen
done before the simulation exists. Internally this is an empty SFF (`{streams: [], metadata: {},
__manual: true}`) rather than a second code path, because every parser here already handles a file
that declares nothing. `__manual` is what keeps it honest — the tab reports no file, the session
save writes no SFF, and a real file arriving later replaces it.

**Adding rows to a file that left them out (Round 9).** *+ Add a stream* sits at the foot of the
sidebar and under the cost summary grid, alongside the Utilities panel's *+ Add a utility*. A
hand-entered stream gets its own editor card — name, direction, and an optional main component —
and is otherwise an ordinary row: it scales with the plant, carries its own exponent, annualises on
the same hours, prices out of the chemical price library, exports, and reaches the LCA tab. The
name is what the library is searched on, so typing "Methanol" fills $1.414/kg with the library's
own provenance string; renaming re-asks but never overwrites a price already on the row, and the
*Look this name up* button is how you ask for the library's answer on purpose. The component, when
given, declares the stream 100% that chemical and is what resolves a CAS for the LCA tab; leave it
blank for a mixture and the stream travels as mass with no composition attached.

**Hand-entered rows survive a file load.** They exist *because* the file did not have them, so
`beginReview()` carries them — with their categories, prices, flows and review decisions — across a
later upload or a broadcast from the SFF CapEx Matcher, and the file's own streams are added around
them. The same applies to hand-added utility rows. Everything says which rows are yours: a "yours"
tag in the sidebar and the summary, "entered by hand" on the panel, and an `origin` column in both
exports and on every row of the `inventory-update` broadcast.

**Boundary-stream detection.** A stream counts as a boundary stream if either end has no unit
attached — `source_unit_id === "None"` (something enters from outside the process) or
`sink_unit_id === "None"` (something leaves it) — rather than the `roles`/`price` fields the old
`SFF_streams_scrapper.html` assumed, which don't exist in real SFF files (see §6). Internal
stream-to-stream connections are filtered out.

**Direction and classification.** Direction (input/output) falls out of the boundary check above.
Category (Raw Material / Utility / Waste Treatment / Product Sale) is suggested from, in order of
preference: the SFF's own `stream_classification` field, its `stream_type` field, whether the
stream's id is listed in the file's `metadata.feedstockJson`/`productJson`, and finally just the
stream's direction as a last-resort default. Always human-editable via a dropdown.

**Three-tier price suggestion**, each one tried in order and shown with its source so the reviewer
knows how much to trust it:
1. **Direct stream price** — some SFF files (SuperPro-simulation-style exports) report
   `stream.price` directly on the feed/waste/product stream. Most reliable when present.
2. **Chemicals table match** — the stream's dominant composition component (by mass fraction, or
   mass flow if available) is looked up in the file's `chemicals[]` array, which on
   simulation-export files carries `purchase_price_usd_kg` / `selling_price_usd_kg` /
   `waste_treatment_usd_kg` per named chemical. The field used depends on the stream's category
   (a Product Sale stream prefers `selling_price_usd_kg`, etc.), falling back to whichever of the
   three is populated.
3. **Fuzzy utility match** — the stream's display name is fuzzy-matched (same tokenize-and-score
   approach as the SFF CapEx Matcher's equipment matching) against `utilities.other_utilities` /
   `power_utilities` / `heat_utilities`, which is where literature-extraction-style files
   (e.g. the MSW example) put pricing instead of on the stream itself. Only mass-basis prices
   ($/kg, $/MT, ...) are usable this way — energy-basis prices ($/MWh, $/Gcal) are still surfaced
   in the candidate list for visibility, but flagged as not directly usable against a mass flow and
   never auto-applied.

**Tier 4 — the toolkit's own chemical price library (`chemical_prices.json`, Round 6d).** The
first three tiers all read the SFF. On real files that is frequently not enough: the
MSW-to-methanol example prices neither its feedstock nor its product, which are the two numbers
NPV, IRR and MSP turn on. This tier is the only one that can price a stream the file left alone.

- **Last on purpose.** Anything the file says about its own process outranks a generic published
  price.
- **Matched by CAS first**, then exact name or alias, then fuzzy. SFF files carry
  `chemicals[].registry_id`, and CAS is exact where names are not.
- **19 entries, deliberately.** A short list that is all sourced beats a long one that is half
  guessed. Every price carries a source, a URL or DOI, a basis year and a unit that normalises to
  `$/kg`; entries that could not be sourced ship with `price: null` and still *match*, so the
  reviewer is told to get a quote rather than left with silence.
- **Reported on its published year, never escalated.** Each entry carries a `basis_year` and the
  price is used exactly as published; the candidate card, the suggestion source and the Appendix
  all print e.g. `2007 basis` and stop there. No age is computed from it — the year is the fact,
  and whether it is still good enough is the reader's call. Nothing escalates, and nothing should
  escalate against **CEPCI**, which tracks equipment construction cost, not chemical markets. The
  right index is a chemical PPI (BLS/FRED), which this toolkit does not carry; add one on the
  `utility_costs.json` series pattern if escalation is ever wanted. Eight entries currently sit on
  a 2007 basis.
- **A negative price is a payment received.** Municipal solid waste is the case that matters: a
  waste-to-X plant is paid a tipping fee to take its feedstock, so its feedstock cost is negative.
  The same EREF figure appears twice with opposite signs — MSW feed `-$62.61/MT`, ash to landfill
  `+$62.61/MT`.
- **Fuzzy matching here is stricter than elsewhere** — threshold 0.45 against the utility
  matcher's 0.3, and aliases under three characters are exact-only. Both rules exist because the
  first cut priced three internal *syngas* streams as natural gas (the alias `"ng"` is a substring
  of "syngas") and `Flue gas` as natural gas on the shared token "gas". That is the §7 failure
  mode — "finds something plausible-but-wrong", not "finds nothing" — and it would have been a
  silent, confident error in a total.

If none of the four finds anything, the price field starts blank (never a silent zero) so the
reviewer knows it needs manual entry — the dashboard's "unpriced streams" counter tracks this.

**Unit normalization.** Every stream's mass flow is converted to kg/h and every price to $/kg
before multiplying, regardless of what units the SFF happened to report either in (`MT/D`, `$/MT`,
`lb/hr`, etc.) — this fixes a real bug in the old tool, which multiplied raw numbers together
without checking units matched at all. A unit-converter widget (ported from the same engine the
SFF CapEx Matcher and Estimator use) is available as an escape hatch for mass-flow units the built-in
table doesn't recognize (e.g. an unusual alias); molar and volumetric flows can't be priced without
a density or molecular weight this tool doesn't have, so they're left unconverted and flagged
rather than guessed at.

**Per-stream review panel** shows stream properties (mass flow, temperature, pressure, composition
by mass %), the suggested category and price with its source, all price candidates found (direct /
chemicals-table / fuzzy-utility) as clickable options, the live per-stream cash flow, and the unit
converter. **Two things you can do with each stream:** accept/edit the suggestion and **Confirm**,
or **Skip** and come back later — there's no bootstrap/manual-entry distinction here since every
boundary stream already exists in the file; there's nothing to fabricate a whole new entry for.

**Live dashboard** (always visible, computed from every stream's *current* state — not gated on
being marked "confirmed", since the point is to see the running total as you go): total input cost
($/hr), total output revenue ($/hr), net OpEx cash flow ($/hr), and a count of unpriced streams.

**Exports** (all downloadable any time): `stream_pricing.json` and `stream_pricing.csv`, both with
one row per boundary stream — id, display name, direction, category, price (value + unit +
source), normalized mass flow (kg/h), cash flow ($/hr), and review status. There's no
`stream_alias_map.json` yet carried forward across sessions the way `unit_alias_map.json` is for
the SFF CapEx Matcher — see §7. (This tool also has a second mode, **Utility Duty & Annual OpEx** — see
§5.5 — reached via the mode-switch buttons that appear once a file is loaded.)

### 5.5 Unit Duties (`streams_matcher.html`, fourth view) — Round 6f

> *This section replaces the pre-Round-4 "Utility Duty & Annual OpEx" mode, which no longer
> exists. Its annualisation and maintenance/labour settings moved to Plant Economics (§5.6); its
> duty parsing was dropped in the rewrite and is what this view restores, correctly.*

**The problem.** §5.5b gives every declared utility a price. It gave them an **amount** from one
source only: a plant-level figure stated on the `utilities.*[]` entry. On the MSW-to-methanol
example **none of the seven utilities carry one**, so all seven priced at nothing — while the
Aspirin example carries thirteen per-unit duties that nothing read.

#### Where an amount can come from

A per-row dropdown in the Utilities panel, drawn only where there is an actual choice:

| Source | What it is |
|---|---|
| **SFF plant total** | The aggregate the exporting software published on the utility entry. |
| **Per-unit duties** | The sum of that utility's duties across every unit, converted into this row's price basis. |
| **Typed here** | A hand override. |
| **Auto** *(default)* | Plant total if the file states one, else the duty roll-up. |

Auto prefers the plant total because an aggregate the exporter published outranks one this tool
derived. Simulation exports usually have no plant total, so the roll-up is the only thing that can
fill the row; literature-extraction files are often the reverse.

#### Units — and how to check them

The Utilities panel multiplies amount × price with **no conversion of its own**. Every conversion
happens on the way in, and all of it is printed.

- **The unit comes from the field name or from an explicit `units` key — nothing else.**
  `peak_heating_duty_W` → W; `duty_MMBtu_h` → MMBtu/h; `..._kJ_h`, `_MJ_h`, `_GJ_h`, `_kcal_h`,
  `_Mcal_h`, `_Gcal_h`, `_kW`, `_MW` likewise. A bare `_J` is **rejected** — joules are not a
  rate — and an unsuffixed field with no `units` key is **listed and excluded**, never assumed.
- Everything normalises to **watts**, then into the denominator of the matched utility's own price:
  a `$/kWh` price needs kW; a `$/kg` price needs kg/h, bridged by that utility's `mass_to_energy`
  (J/kg). No `mass_to_energy` → the row is left unpriced and says why, rather than inventing a
  mass flow from an energy duty.
- **Produced** duties (`utility_production_results`, e.g. exported steam) are **netted**, not added.

The chain is printed under the amount box and again in the roll-up table:

```
401,070.79 W × 3600 s/h ÷ 2,107,407.9 J/kg = 685.1 kg/h
```

#### Attribution

`hx_agents` names the agents serving a unit; it does not say which duty each one serves. So:

1. **Role filter** — a `..._heating_duty_W` field can only be attributed to an agent whose own
   `thermal_role` is `heating`, and likewise for cooling. Badged `exact` when one survives,
   `only candidate` when the unit declared exactly one agent of that role.
2. **Power duties** bypass `hx_agents` entirely and go to the `power_utilities` entry.
3. **Two agents of the same role on one unit** → badged `ambiguous`, a dropdown offers the declared
   agents, and the duty is **excluded from the roll-up until answered**. The file does not state
   the split; the tool does not invent one, and does not silently take the first.
4. **No agent of that role declared** → `unattributed`, same dropdown.

`operations_using_agent` is a **checksum, not a tie-breaker**: it counts operations per agent
(Steam: 4 of 4) and reconciles against what was parsed. It is silent on how a shared duty splits.

#### The view itself

Four pills — duties found, across how many units, **how many need an answer**, how many have no
usable unit — then two tables: **Rolled up per utility** (net watts, the figure in the price basis,
the conversion line by line) and **Every duty, unit by unit** (unit, field, the reported value in
the file's own unit, the watts, the utility, the badge and a sentence of why). Export:
`unit_duties.csv`, including which rows a reviewer answered by hand.

Answering an ambiguous duty re-derives every Utilities row still on Auto or Per-unit duties.

### 5.5b Utilities panel (`streams_matcher.html`, third view) — Rounds 4–6

*Not documented before Round 6; this section is the catch-up.*

Boundary streams (§5.4) cross the system boundary. **Electricity and cooling water do not** — they
have no stream to hang a price on, so before this panel existed they had nowhere to be stated,
nowhere to be checked, and no path into the OpEx total except a hand-typed lump sum in Plant
Economics. The panel gives each declared utility an amount, a price, and a stated basis.

**Where the amount comes from is §5.5** — a plant-level figure the file states, or the per-unit duty
roll-up, or a typed value, chosen per row. Everything below is about the **price**.

**Where a price comes from**, in strict descending precedence, with each row badged:

1. **MANUAL** — you typed it. Nothing recomputes over it.
2. **SFF FILE** — the uploaded file priced this utility itself.
3. **MEASURED** — an EIA annual series for the selected basis year (electricity, natural gas).
4. **CORRELATED** — Ulrich & Vasudevan (2006), `C = a·CEPCI + b·C_fuel`, for the utilities nobody
   publishes a market price for: steam, cooling water, refrigeration, treated water, wastewater, air.

Everything lives in **`utility_costs.json`** (series, coefficients, validity ranges, notes). Nothing
is hardcoded in the page, and `test_utility_pricing.js` reproduces Ulrich & Vasudevan's own worked
examples before any of it reaches the UI.

**The year factor is the whole point.** The correlation is evaluated against the *same* CEPCI the
CapEx Estimator is escalating equipment with (relayed as `cepci-update`, and from Round 6 that
follows the Estimator's year dropdown), and the EIA gas price for the same year. Move the year and
capital cost and utility cost move together, instead of one being stranded in 2013 while the other
sits in 2026.

**Choosing between the file's price and the toolkit default — new in Round 6.** A row that has both
gets a dropdown:

| Choice | Behaviour |
|---|---|
| *Auto — SFF file first* (default) | The precedence above, unchanged. A price the file states is usually the project's own negotiated figure. |
| *SFF file* | Force the file's price even where a library default exists. |
| *Toolkit default* | Force the EIA/U&V value **even though the file priced it**. |

The third option is the one that needed adding. SFF files are frequently written against a different
basis year or region, or carry a placeholder the modeller typed — and until now the only way to set
that aside was to delete it and type a number by hand, which loses the derivation. Choosing the
default keeps the correlation **live**, so it still moves when the basis year or the CEPCI moves.
The row says what it set aside and by how much (`the SFF file prices this at $0.15 (1.74× the
default shown)`), because the disagreement is the interesting part. A forced choice that cannot be
honoured falls back and says so rather than blanking the row — a null price drops silently out of
the total, which is worse than the wrong source. Choosing a source releases any manual price on that
row, otherwise the manual value would keep winning and the dropdown would appear to do nothing. A
**Price source — all rows** control in the pill row sets every row at once as a starting point;
individual rows can still be changed afterwards.

**Which SFF field the price is read from — Round 6b.** A price is taken from the first of
`price`, `regeneration_price`, `heat_transfer_price` that the entry carries. This matters more
than it sounds: **a heat utility in a BioSTEAM-style export generally has no `price` field at
all**. Until Round 6b the panel read only `price`, so in the MSW example cooling water
($3.6509/Gcal via `heat_transfer_price`) and LP steam ($5.7/MT via `regeneration_price`) both
showed as unpriced, fell through to the correlation, and — because the panel thought there was no
SFF price — did not even offer the source picker above.

The field is named on the row, in the picker's option label and in the CSV, because the three are
not interchangeable. `regeneration_price` is the cost of *regenerating* the utility and an
exporting plant may state it as a **credit** (the MSW file's snippet reads "LPS credit
(@$5.7/MT)"). A row priced from that field carries a warning saying so. The number is worth
showing — it is the only figure the file gives — but whether it belongs in the cost column is the
reviewer's call, not the parser's.

**Hand-matching a row to a library utility — also Round 6, and load-bearing.** `defaultsFor()` needs
a 0.3 name similarity to claim a match, and SFF files name utilities however the source software felt
like: `"Std Power"`, `"CW"`, `"LPS"`. `"Std Power"` does not clear the bar against `"Electricity"`.
Without a way to point such a row at a library entry by hand, **no library default exists for it at
all**, so the choice above could only ever be offered on conveniently-named utilities. Unmatched rows
therefore get a dashed *"No library match — pick one…"* dropdown; picking one produces the default
price, the correlation, and the source picker. A hand match is recorded as such (`matched to
«electricity» (by you)`, and `(hand-matched)` in the CSV) and outranks the fuzzy match on a rebuild —
a reviewer's answer is a stronger claim than a 0.31 similarity score, and the export should not
present the two as the same thing.

**Capacity terms.** Several U&V correlations depend on throughput — cooling water at 0.01 m³/s costs
about 30× what it costs at 10 m³/s — so the amount feeds the capacity term. A **system capacity**
column pins the *utility system's* size when the plant draws from a larger shared one; in U&V the
capacity variable is the system's, not the consumer's, and their alkylate example only reproduces at
the site's 10 m³/s rather than the module's 0.10 m³/s draw. If the amount's unit cannot be converted
to what the correlation expects, the price is left unset and the row says why, rather than producing
a plausible number from an assumed flow.

**Why electricity does not use the correlation.** At today's inputs U&V returns ≈ $0.15/kWh against
an EIA industrial average near $0.086/kWh. That is not an extrapolation artefact — at the paper's own
mid-2000 basis it returned $0.091/kWh against ≈ $0.046/kWh, so the offset is in the coefficient. Their
`a` term is a modelled delivered cost carrying capital recovery, a different quantity from a national
average that very large consumers pull downward. Electricity and gas use the measured series, with the
correlation displayed beside them so a reviewer can see the disagreement rather than have it quietly
resolved for them.


### 5.6 Plant Economics (`plant_economics.html`) — new 2026-09-10

Fifth tab. Owns exactly the two layers the other four never covered: everything **above** the
bare-module sum, and everything **below** the variable-cost total. Six panels:

- **Capital cascade** — all three conventions computed simultaneously from the same `Σ C_BM`, with the
  selected one broken out line by line and a headline "spread across conventions" figure:
  - *Turton*: `C_TM = 1.18 Σ C_BM`; `C_GR = C_TM + 0.50 Σ C_BM(base)`
  - *Seider*: `C_TBM → C_DPI → C_TDC (×1.18) → C_TPI → C_TCI`, working capital solved implicitly
  - *Towler–Sinnott*: `ISBL → +OSBL (40%) → +engineering → +contingency → FCI → +WC`
  Each reports **which rung it is treating as FCI**, because the three don't agree and reporting a
  bare "FCI" without saying which is how two partners' numbers end up incomparable while both look
  right. Flags bootstrapped rows (→ raise contingency), rows with pressure uncorrected, and the CEPCI
  escalation window.
- **Operating labour** — Turton's `N_OL` correlation off the `equip_class` census, with the class
  breakdown shown and both counts editable. Reproduces Turton's own worked example exactly
  (N_np = 11, P = 0 → N_OL = 2.97 → 14 operators), which is the strongest available check that the
  coefficients in `costing_basis.json` are right.

  > **Round 19 — an empty plant is not staffed.** The correlation is
  > `N_OL = √(6.29 + 31.7 P² + 0.23 N_np)`, and it has an **intercept**: with nothing at all on the
  > flowsheet it still returns √6.29 ≈ 2.51 operators/shift, ×4.5 shifts → **12 people**, and a
  > six-figure `C_OL` that made the cost of manufacture positive for a plant with no equipment in
  > it. That intercept is where Turton's fitted line crosses the axis, not a claim that an empty
  > site needs a crew — the correlation was fitted over real plants and is not defined below them.
  > The panel now treats labour as **not applicable** when nothing is costed and no crew has been
  > typed: `C_OL = 0`, the reason stated on screen, and the intercept named rather than hidden. A
  > crew **typed by hand** against an empty flowsheet is still costed, because a stated number is
  > not a guess — the panel just says there is no flowsheet behind it. The capital panel says the
  > matching thing: with `Σ C_BM = 0`, every capital-keyed figure below it is zero. Checked in both
  > directions in `test_round19_ui.js` §3, so the flag cannot start crying wolf.
- **Cost of manufacture** — variable costs taken as-is from the SFF OpEx Matcher tab (already better
  grounded than any factored shortcut, so nothing re-derives them), a manual consumables table, then
  the convention's fixed-cost lines. Turton's closed form, Seider's cost sheet, or Towler–Sinnott's
  variable/fixed split.
- **Cash flow & MSP** — MACRS 5/7 or straight line, tax, discount rate, construction profile, WC
  recovery → NPV, IRR (bisection, returns `null` rather than a confident wrong number when there is no
  sign change), payback, and an iterative minimum-selling-price solve. **Strips the convention's own
  depreciation line before using COM as a cash cost** — double-counting depreciation is the commonest
  error in a hand-rolled DCF built on a factored cost sheet, and the panel says so on screen.
- **SFF-reported totals** — the source software's own NPV/IRR/ROI/payback/investment figures from
  `metadata`, compared against this toolkit's bottom-up build, flagged above 30%, **never folded into
  any total**.
- **Capital breakdown & scale sensitivity** — *new in Round 6.* Two questions the cascade cannot
  answer on its own: **which equipment is the capital**, and **what happens at a different plant
  size**. Both charts used to live only in the CapEx Estimator, where they could only be read
  against installed cost; here they are read against the same FCI the rest of the page is built on.
  - The *breakdown* is drawn from the per-row detail `costs-update` already carries — the same array
    the traceable report's Equipment section is built from — grouped by line item, category, or
    `equip_class` (the last so capital and headcount can be split on the same axis). A table beside
    the pie gives share and **cumulative** share, and a footnote states how few groups carry 80% of
    the capital, which is the question a pie answers badly.
  - The *sweep* is **computed by the Estimator on request**, not here. This page holds each row's
    result, not the correlation behind it, so it cannot re-evaluate a pump at three times the flow.
    It sends `request-scale-sweep`, the Estimator re-runs every correlation at each scale factor, and
    the (scale, installed cost, `Σ C_BM`, `Σ C_BM`-at-base) curves come back. One cost model, one
    evaluator.
  - The cascade is applied **at draw time**, point by point, with the convention and factor values
    currently selected — so switching convention or nudging a factor re-plots the same sweep with no
    second round trip. Plot FCI, TCI, or raw installed cost.
  - The fitted exponent is **translated**, not just printed: `n < 1` is reported as an economy of
    scale with what doubling the plant actually costs, and compared against the classic six-tenths
    rule. `n ≈ 1` is reported as *no* economy of scale, which usually means parallel trains are being
    added rather than bigger equipment.
- **Audit & export** — `tea_report.json`, in which every figure carries its convention, factor values,
  factor overrides, CEPCI basis and target, provenance tier, and the convention spread as an explicit
  uncertainty block. Plus a flat cost-sheet CSV. There is an empty `lca` stanza so the schema is
  stable when emissions accounting arrives.

**Bridges.** Consumes `costs-update` from the Estimator (purchased / bare-module / base-condition sums
plus the per-row detail and the `equip_class` census, sent as one fat message so a consumer can never
hold a cost sum from before an edit alongside a census from after it), `scale-sweep` in answer to its
own `request-scale-sweep` (Round 6), `opex-update` from the SFF OpEx Matcher (net
`$/h`, sign-flipped to a cost), and `sff-file-loaded` for the metadata card only — it deliberately does
**not** re-parse units or streams, since the two tools that own that would then have a third parser to
drift from. Writes nothing back. Works standalone; the linked fields just become manually editable.

#### Cost and revenue are two fields (Round 14)

**Variable OpEx ($/h)** carries the **gross** cost — raw materials, waste treatment and utilities —
and **Annual revenue ($/yr)** carries the product sales. Both are filled from the OpEx tab's
`opex-update` broadcast, and both keep a hand-typed value.

Before Round 14 one netted figure did both jobs: Variable OpEx was filled from `netPerHour`
(cost **minus** sales) and Annual revenue was left at zero. That made the cost of manufacture
*negative* on any plant whose product was priced, zeroed the cost lines that are charged against
sales (selling, admin, R&D, marketing), and made the minimum selling price solve to $0.000. **Every
COM, NPV, IRR and MSP figure produced before that fix is wrong**; a session saved earlier re-opens
corrected and the Cash Flow panel says so once.

#### The minimum selling price and its denominator (Round 13)

MSP is a cost divided by an output. The output comes from the OpEx tab's stream slate, not from a
box somebody types into:

- **The picker** (*Product stream the price is solved for*) lists every output stream with a flow,
  ranked by annual revenue, plus *all Product Sale streams combined* and *enter by hand*. The
  default is the **most profitable single stream** — not the sum of the slate, because a price
  divided by the combined mass of everything a plant sells is a price for a basket nobody buys.
- **Everything about the choice is stated** under the panel: which stream, its kg/h at the current
  plant scale, the hours, and why it was the default. A stream that is not categorised Product Sale
  can be chosen — a miscategorised product is exactly what the picker is for — and is flagged.
- **Typing in the production box** switches the picker to *Enter by hand*, and a hand-typed figure
  is never overwritten by a fresh slate. A chosen *stream*, by contrast, follows the plant scale.
- **The result names its product**: "$1.101 per kg of Methanol product".

The solve replaces **only the priced-for stream's** revenue with `price × production`. Other
priced products go on earning at their own prices, and the panel says so — dropping them would
overstate the price the main product has to fetch. Choosing *All Product Sale streams combined*
reprices the whole slate, so nothing is held back.

**The capacity sweep uses the same pick (Round 15).** `request-opex-sweep` carries the chosen
`productId`, and the OpEx tab answers with that stream's own kg/h and the other products' $/h **at
every point on the grid**, so the swept price is divided by the same thing the single-point price is
and the by-product credit moves with the plant. Before Round 15 the sweep divided by the whole
Product Sale slate and credited the by-products at their un-scaled revenue — two ways of answering a
different question from the card above it — and it computed the price without storing it, so the
column was empty at every scale. The chart now states the basis under the curve, and a hand-typed
production figure, which names no stream, is carried across the sweep in proportion to the
flowsheet's own output and flagged as such.

### 5.6b Cost Breakdown panel (`plant_economics.html`) — Round 12

The Cost of Manufacture panel's own figures, regrouped into five buckets and drawn. **Nothing here
is computed that that panel does not already compute** — the buckets are required to sum to its
total exactly, and the panel prints a flag if they ever do not.

- **The buckets** are Input streams, Utilities, Labour/supervision/overheads, Capital-derived, and
  Revenue-linked & other — assigned by each cost line's own `basis` string (`C_TDC`, `ISBL`,
  `labour + supervision`, `sales revenue`), which is what the factor is multiplied by and therefore
  the honest statement of what drives it. Round 12 had a sixth, a product-sales credit drawn below
  the baseline; Round 14 took sales out of the cost, so there is nothing to credit. A bucket can
  still go below the line — a feedstock the plant is *paid* to take, like the MSW tipping fee, makes
  Input streams itself negative.
- **Three charts.** Composition by convention (all three, side by side — this page never picks one
  for you, and the chart shows *which bucket* they disagree about); the selected convention's
  buckets ranked; and cost against revenue on one axis, with the operating margin stated.
- **A table of the same figures** sits below them — the copy-out view, and the way to read the
  panel without relying on colour.
- **Five buckets is a cap, not a preference.** The categorical palette is validated against
  colour-vision criteria for adjacent pairs, and a sixth hue could not clear the floor against the
  five without one of them moving. See `HANDOVER.md` → Round 12 before changing a colour.

### 5.7 Plant capacity — the capacity bar (`equipment_toolkit.html`) — Round 7

The strip under the tabs. One number, and every tab is now a function of it.

**What it holds.** A capacity in engineering units (MT/D of feedstock), a dimensionless scale
factor, and a line saying where the flowsheet's own capacity was derived from. `capacity = base ×
scale`; the tabs only ever see the scale.

**Where the base comes from.** The SFF OpEx Matcher derives it from the streams the file itself
declares in `metadata.feedstockJson`, and reports it up. It is *not* parsed in the wrapper,
because that tab is the only stream parser in the codebase and a second one is how the two
matchers drifted apart in the first place. When the file declares no feedstock, the derivation
falls back to the streams the reviewer categorised `Raw Material` **and the basis line says so** —
a capacity from a fallback and one from the file's own declaration are different claims.

**When nothing can be derived**, the capacity box is disabled and says so; the scale box still
works. A bare multiplier is an honest thing to sweep. An invented "1,000 MT/D" would not be.

**What moves with it.** Equipment sizes (the Estimator's existing Global Scale Factor, now driven
from here), boundary-stream mass flows, utility amounts, product output, and therefore every cash
flow, the cost of manufacture, the DCF and the life-cycle inventory.

**Ownership, stated because it is the thing not to change casually.** The wrapper owns the scale
and broadcasts `scale-update`. Tabs consume it and never write back — with one exception: the
Estimator's own scale box stays editable and announces `scale-change`, which the wrapper adopts
and relays to everyone *except* the Estimator. That "except" is what makes two-way editing safe
without a guard flag.

**Per-row exponents.** Flow ∝ scale^n, n = 1 by default, editable per stream (cost summary grid)
and per utility (Utilities panel). n = 1 is right for a material balance; lower it for a purge, a
catalyst make-up, a fixed heel or a timer-driven regeneration. Blank reverts to the default; **n =
0 is a real claim** (does not move with capacity at all) and has to be typed on purpose. Every
exponent appears in the exports.

**Design flow vs. flow at scale.** The cost summary shows both, as separate columns, and the
editable box is the *design* one. This is deliberate: folding scale into the edited value would
mean typing in that box at scale 2 wrote back a doubled number, compounding on every change.

### 5.8 Capacity & Surrogate panel (`plant_economics.html`) — Round 7

Sweeps the **whole chain** against capacity, and fits the surrogate the supply-chain model
consumes.

**Two requests, two evaluators, one x-grid.** This page asks the Estimator to re-cost every row
and the OpEx Matcher to re-price every stream and utility, over the same `(start, end, points)`.
Both build the x-axis with the same log-spacing, so the answers join index for index; a
length mismatch is refused rather than silently paired off. Everything after the join — cascade,
labour, cost of manufacture, cash flow, MSP — is this page's own model at the convention and
factor values currently selected, so changing the convention moves the curves.

**The OpEx half is re-evaluated, not extrapolated.** The OpEx tab moves its own scale variable and
re-reads every priced row. Multiplying today's $/h by the scale factor gives the same answer only
while every exponent is 1 — the assumption the exponent exists to let a reviewer break.

**Curves:** minimum selling price, NPV, IRR, levelized cost of product, cash cost of manufacture,
FCI, Σ C_BM — against capacity or against the bare scale factor, on log or linear axes.

**What the $/kg curves are per kg OF (Round 15).** MSP and LCOP are both divided by the **picked
product stream**, re-scaled at every point by the OpEx tab, with the other products' sales held as a
credit at that same point; the basis is printed under the curve, the table column reads *Priced
product (kg/yr)*, and both exports carry `priced_product_kg_per_yr`, `product_slate_kg_per_yr`,
`other_product_revenue_usd_per_yr` and `msp_denominator` beside each other. LCOP uses the same
denominator and the same credit as the MSP, so the two differ only by tax and can be read together.
The **surrogate fit for product output is still the whole slate** — it is the plant's output law for
the optimisation model, not a price denominator. The oracle for all of this: at the scale the
toolkit is actually sitting at, the swept MSP and the single-point card are the same plant and must
agree to the solver's tolerance; `test_round15_ui.js` §3 pins exactly that.

**Held constant, and said to be:** operating labour (Turton's correlation counts process *steps*;
the sweep changes their size, not their number), manually-entered consumables, and the financial
assumptions.

**The surrogate.** Capital is fitted on **Σ C_BM, not TCI**, so the optimisation model can apply
the capital cascade analytically and vary contingency, OSBL fraction or working capital without
refitting. Variable cost is fitted **through the origin**, so the capital-fraction fixed costs
stay analytic instead of being partly absorbed into an intercept. Exported as JSON with the
cascade ratios, the financial assumptions and the basis (convention, CEPCI, hours, price coverage,
how the capacity was derived) — a surrogate without its basis is a table that looks fair while
not being fair.

### 5.8b How a swept curve is fitted — Round 20

Three charts plot a quantity against plant size: the CapEx Estimator's **cost vs. scale** chart
(§5.2), the **capacity sweep** and the **Capacity & Surrogate** panel (§5.8). All three read the
same swept points through one shared fitter, and all three now let you choose how.

| Form | What it is | When it is the right one |
|---|---|---|
| **Power law** `a·x^b` | least squares on `ln y = ln a + b ln x` | equipment capital. It *is* the six-tenths rule, and `b` is a number with a meaning — the panel translates it rather than only printing it |
| **Quadratic** `c₀ + c₁x + c₂x²` | ordinary least squares, linear space | anything with a fixed part and a proportional part, and **anything that crosses zero** — an NPV sweep, an IRR sweep. A log fit cannot see a negative number |
| **Piecewise linear** | a continuous fit with an adjustable number of segments | a surrogate for an optimisation model. An MILP cannot take `a·x^b`; it takes breakpoints |

**The piecewise fit is continuous by construction.** It is fitted on the hinge basis
`1, x, max(0, x−b₁), max(0, x−b₂), …` rather than by fitting each segment on its own, so every
function in its span joins up at the breakpoints. That is not cosmetic: an optimiser handed a
surrogate with a jump at a breakpoint will sit exactly on the discontinuity, and the SOS2
formulation these coefficients are for cannot express one. `test_scaling.js` §6 checks continuity
on data the fit deliberately *cannot* match.

**Breakpoints sit at equal quantiles of x**, not at equal spacing, because every sweep here is
log-spaced — equal spacing would put most of the points behind the first segment. They are drawn
on the chart as dashed verticals, because where the segments meet is the thing you are actually
choosing when you set the segment count, and an MILP built on these coefficients will bind at
exactly those values.

**Two R² figures are reported when they differ.** The power law is fitted in log space and the
other two in linear space, so their R² values are not on the same footing. The summary line prints
the R² in the fit's own space and, beside it, ordinary R² measured on the raw values — which is
the figure to compare across forms. A power law will often show a flattering log-space R² and a
worse linear-space one than a quadratic on the same points; that is a real property of the two
fits, and hiding it would make the picker useless.

**A fit is refused rather than guessed.** Too few points, more segments than points, a power law
over a curve that reaches zero — each says which constraint failed and which number to change.
A log **axis** cannot draw a non-positive point either; those are counted and reported rather than
quietly dropped, and switching the y axis to linear brings them back.

**What reaches the optimisation model.** The surrogate export (`viff-tea-surrogate`) still carries
the power-law `capital` block every existing consumer reads, untouched. When the picker is on
another form, `capital_alt_form` is carried **beside** it with `nodes` — the breakpoints as
`{Q, capital_usd}` pairs, the SOS2 form — and `segments` for a big-M formulation. Q is plant
capacity in the flowsheet's own units, not the scale factor.

### 5.9 Life-Cycle Impacts (`lca.html`) — Round 7

The seventh tab. See §7 item 6 for what it can and cannot say; the short version is that it is a
**gate-to-gate** inventory, it says so on screen, and nine of the ten TRACI categories are blank
rather than zero because their factors are not yet transcribed.

**Four panels.** *Impact Results* (per category, split between direct emissions and purchased
utilities, with a contributor ranking); *Inventory* (every stream and utility at the current
scale, with the fate decision); *Coverage & Gaps* (what fraction of boundary mass was actually
characterised, and an itemised list of what was not and why); *Method & Factors* (every shipped
factor with its source, URL, basis year and derivation).

**The functional unit** is selectable: per operating year, per kg of product, or per operating
hour.

**Fate** is a reviewer decision, not an assumption. Released to air, sent off-site for treatment,
or not an emission — suggested from the stream's own name, and left **unanswered** when the file
gives nothing to go on, in which case the stream is excluded and listed rather than guessed.

**Released streams are characterised by their components**, not by total stream mass. A flue gas
that is 40% CO₂ characterised as if it were all CO₂ overstates by 2.5× and looks plausible. A
released stream with no stated composition is counted as uncharacterised.

**Biogenic CO₂** gets its own line and its own switch. Counting it as zero is an accounting
convention, not a measurement, and on a biomass or waste flowsheet it is usually the largest
single number in the inventory.

#### Contribution charts (Round 12)

Above the Contributors table, built from the same rows and the same category selector:

- **Ranked contributors** for the selected category, coloured by kind (direct emission, purchased
  utility, purchased material) rather than by rank. Past twelve bars the tail folds into one
  "smaller sources" row; all of them remain in the table below.
- **How much of each category is the utilities** — the emission/utility/material split as
  **shares**, because the ten categories are in ten different units and one value axis cannot carry
  them honestly. A category with no populated factor is not drawn at all: a blank is an absence,
  not a zero.

**Equipment is not in these charts, and that is a decision.** Embodied impact needs an emission
factor per material *and* a mass per unit. `impact_factors.json` carries no equipment factors, and
192 of the 196 entries in `equipment_library.json` are sized on area, power or flow rather than
mass — so for 98% of the library there is nothing to multiply even if the factors existed.
`TODO.md` → G0d records what closing that would take.

### 5.10 Parameters & basis file (`appendix.html`) — Round 8

The Appendix stopped being read-only. Every number the toolkit takes as given is now listed,
searchable and editable in one place, and a **basis file** carries a lab's own values between
sessions and between people.

**What is listed.** Every numeric leaf of `costing_basis.json`, `utility_costs.json`,
`chemical_prices.json` and `impact_factors.json` — 371 parameters — keyed by its JSON path, with
its unit, its source and the shipped value beside it. Filter by file, search by name or path, or
show only what you changed.

**What is not, and why.** `equipment_library.json` is generated from `equipment_library.xlsx` by
`build_equipment_library_json.py`, and rule 3 of this codebase is that generated files are never
hand-edited — an override made here would be silently lost on the next rebuild. Edit the workbook.
Definitions are prose. Both stay read-only on their own panels.

**An edit is live everywhere, immediately.** Writing a value mutates the same object the other
panels render from, and the Appendix broadcasts `basis-update` to every tab: Plant Economics
re-runs its cascade, the OpEx Matcher re-resolves every utility and chemical price, the LCA tab
re-characterises. A per-factor override made on the Plant Economics panel is deliberately kept —
that is a decision about *this flowsheet*, while the basis file is a decision about the lab's
defaults, and the narrower one wins.

**Nothing is saved until you download it.** The page says so. There is no storage layer; the basis
file is the storage layer.

**The basis file is a full snapshot, not a diff** — and the trade is worth knowing. Self-contained
means a colleague gets exactly the numbers you ran on. The cost is that it freezes files that are
meant to be maintained: a snapshot taken today keeps today's CEPCI table and today's prices
forever. `changed_from_shipped` is what keeps the decisions legible inside it — it records every
path that differs from the shipped defaults with both values, so a reader can see what was decided
and a later session can re-apply just those onto newer files. **Read that block first.**

**Reset** returns every tab to the shipped defaults.

### 5.6c Owning a utility plant, or buying the utility — Round 22

**A correlated utility price already includes the utility plant.** Ulrich & Vasudevan's prices are
`C = a·CEPCI + b·C_fuel`, and their `a` term is a modelled *delivered* cost carrying capital
recovery and O&M. For cooling water at a normal plant rate the split is about **96% plant, 4%
fuel**; every correlated row on the Utilities panel now prints its own split for that reason.

So there are two consistent ways to cost a cooling loop, a steam plant or a refrigeration package,
and they must not be mixed:

| | CapEx Estimator | OpEx Utilities panel |
|---|---|---|
| **Buy the utility** | no equipment row | correlated U&V price, as shipped |
| **Own the plant** | the tower / boiler / package as a costed row | override the price to a **marginal** one — the `b·C_fuel` term plus make-up water and chemicals |

Doing both pays for the equipment twice: once as capital through Σ C_BM → FCI → the convention's
capital-fraction fixed costs, and again inside every unit of utility bought all year.

**Plant Economics flags it**, on the Capital Cascade panel, because it is the only tab that
receives both the equipment list and the utility slate. The flag names the equipment, names the
utility, and prices the overlap — *"of $1.01M/yr of utility about $970k/yr is paying for equipment
this page has already capitalised"* — then states the two treatments without choosing between
them. It is a flag, not a block: which one is right depends on the plant, and only the reviewer
knows.

> **A measured price is never flagged, and that is deliberate.** Electricity and natural gas come
> from the EIA series — a market price for something bought across the fence, containing no plant
> of this plant's. A site that builds its own cogeneration unit is doing something real. A warning
> that fired there would be a false alarm, and a false alarm teaches people to dismiss the warning
> that is right.

### 5.10b Impact Factors (`appendix.html`, sixth panel) — Round 21

`impact_factors.json` used to be visible only through the **Parameters & Basis File** table, which
is a generic walker: every numeric leaf of every data file, named from whatever sibling field
carries a label. Right for a flat list of priced chemicals; wrong for this file, where the number
sits two levels below the thing it belongs to —
`characterization_factors[2].factors.gwp.value` — and the container of `.value` has no name at
all. The result was thirty-one entries' worth of rows called **"gwp › value"**.

The panel is shaped like the file instead:

- **One row per entry**, grouped into *Substances released*, *Purchased utilities* and *Purchased
  materials*, which is the file's own three-list structure.
- **One numeric column per category that somebody populates.** The rest fold into a single header
  saying how many are empty. The fold is **computed from the data** — a category filled in later
  earns its own column with no edit to this page. Today that is one column and a strip labelled
  "9 empty".
- **A derived headline**, so it cannot go stale: *"10 of 310 possible factors are populated (31
  entries × 10 categories), and every one of them is Global warming."*
- **Open a row** for the argument behind the number: the source resolved to its full citation and
  URL, the basis year, the derivation where a factor was derived rather than quoted, the tier, and
  the note. An entry with **no** factor says so, and says what the LCA tab does with it — reports
  it as uncharacterised and counts its mass in the uncounted total, rather than treating it as
  zero impact.
- **The ten categories**, each with its unit, whether it is global or regional, and how many
  factors populate it.
- **Every source**, with its citation, URL and how many factors cite it.

**Values are editable here, through the same writer the Parameters table uses.** One path, so an
edit made on either panel means the same thing: it changes the live object the Life-Cycle Impacts
tab reads, marks the parameter as changed, keeps the shipped value for revert, and travels into
the basis file.

> **What this panel is for.** It changed no number. It made one fact legible that was always true
> and always buried: nine of the ten categories are empty, and 21 of the 31 entries carry no
> factor at all — including **cooling water**, which is why a plant with a cooling loop reports a
> GWP that is entirely its purchased electricity. Set **Show → only entries with none** and the
> panel prints the to-do list itself.

### 5.10c Parameters & Basis File (`appendix.html`) — rebuilt in Round 22

Every number the toolkit computes with, from all four data files, in one editable list. Round 22
rebuilt how each row is described, for the same reason Round 21 rebuilt the impact view: a bare
number is unreadable.

| Column | What it answers |
|---|---|
| **Parameter** | the thing that owns the number — the substance, the utility, the chemical, the convention line — found by walking the path and keeping the **last named ancestor**, because in two of the four files the number does not live beside its name |
| **Type** | what kind of number it is, read off the path shape: capital factor, fixed-cost factor, labour model, pressure factor, CEPCI index, utility price model, chemical price, emission factor. `0.23` is a labour coefficient, a material factor or a characterisation factor depending only on where it sits |
| **Value** | editable, marked when changed, revertible to the shipped figure |
| **Unit** | or *dimensionless*, said explicitly rather than left blank |
| **Basis year** | the column that decides whether two numbers can be compared at all. A 2010 correlation and a 2023 emission factor are both perfectly good and are not on the same footing |
| **Shipped / Source** | what it was, and where it came from |

Rows are grouped by type, and the grouping survives a filter. An impact factor's type names its
**category** — `Emission factor — GWP` — since a kg CO₂ eq and a kg SO₂ eq are not comparable
quantities.

**What is deliberately not listed.** A *basis year* is a fact about provenance, not a number to
tune; editing 2021 into 2024 would move nothing else. A published *range* is the guard rail on the
value beside it and already appears there as "outside the published range" — listing it as two
more editable rows per factor took the table from 341 rows to 243 for no gain. Neither is excluded
from the downloaded basis file, which carries whole files.

### 5.11 Export all — Round 23

The toolkit has fifteen download buttons across five tabs, and they all still work. **Export all**,
in the wrapper's top bar, is one place that lists every one of them at once — which tab it comes
from, what it contains, how big it is — with *Download selected as a .zip* at the bottom.

The bundle includes a `MANIFEST.txt` naming the SFF file, the plant scale and its basis, and the
equipment row count, plus one block per file saying which tab produced it and what it is. It also
lists what was **not** included, and why.

**The files are the same files.** The panel does not rebuild anything. It asks each tab to run its
own export buttons with the bytes handed back instead of saved, so what you get is byte-identical
to what you would get by clicking. That is worth knowing because it means the two routes can never
disagree: there is one implementation of each export, and the panel is a caller of it.

**Greyed rows say why.** An export that needs something you have not done yet reports the reason
it would have given you at the button — *"Run the capacity sweep first"* — and one that would write
an empty list says so rather than handing you a plausible-looking `[]`. A row with no bytes cannot
be selected.

**Two things it can legitimately say.**

- *"this file carries no per-unit duties — nothing to write"*: a fact about the flowsheet, not
  about you.
- *"nothing recorded yet — it would be an empty list"*: the matcher's three exports, before any
  unit has been confirmed.

The zip is written with no compression (STORE), deliberately: forty lines with no dependency and
no failure mode, to compress text your filesystem compresses anyway.

## 6. Design decisions & lessons learned

*Read this before re-litigating a choice — it was probably made deliberately, for a reason
recorded here.*

- **Browser HTML is the primary interface, not a Python GUI.** Decided explicitly. Python's role
  (if/when built) is a batch/parsing pipeline, not user-facing.

**Round 6 (2026-09-12) — five of these are worth not re-litigating:**

- **A chart drawn on two pages is one builder, not two.** When Plant Economics needed the cost
  breakdown and the scale sweep, the cheap move was to copy the SVG code across. Two implementations
  of the same chart is two implementations to fix, and the first time one is fixed the other starts
  disagreeing with it — with nothing on either page telling the reader which is wrong. Same argument
  that produced `toolkit_format.js` in Round 5 and split `buildDcfChart()` from its DOM wrapper. The
  builders are now in `toolkit_charts.js`, pure: numbers in, SVG string out, no DOM, no globals.
- **One cost model, one evaluator — the consumer asks, it does not re-derive.** Plant Economics holds
  each row's *result*, not the correlation behind it, so it cannot re-cost a pump at three times the
  flow and must not pretend it can. It sends `request-scale-sweep`; the Estimator, which owns the
  library, the material and pressure factors and the parallel-train logic, does the sweep and returns
  curves. Shipping the correlations to a second page would have created exactly the drift the
  architecture is built to prevent.
- **Repaint after the state moves, inside the mover.** The matcher rails lagged one row behind the
  detail panel because five call sites wrote `renderSidebar(); goNext();`. Fixing the call sites would
  have fixed it until someone added a sixth. `goNext()` repaints, so every caller is covered.
- **A control offering one option is worse than no control.** The utility price-source dropdown only
  appears where both a file price and a library default exist. Where they do not, the row gets the
  dropdown that *would* help — the hand-match picker — instead. This is also what made the feature
  work at all: `"Std Power"` scores below the 0.3 fuzzy threshold against `"Electricity"`, so without
  hand-matching there was no library default to choose, and the feature would have worked only on
  conveniently-named utilities.
- **Removing something from the nav is not the same as making it a later stage.** Round 4 pulled
  Material of Construction out of the sidebar so people would stop treating it as an alternative to
  picking a pump. That was right about the order and wrong about access — it left a reference sheet
  reachable only by walking a full equipment tree. Round 6 puts it back under its own *Reference*
  heading, below the equipment sheets, with the stage-2 relationship stated in the nav itself. The
  ordering lesson is kept by *where it sits and what it says*, not by making it unreachable.
- **The equipment library's master copy is the Excel spreadsheet**, not the JSON. JSON is always
  generated, never hand-edited. This was chosen so VIFF collaborators without a JS/JSON background
  can maintain it.
- **No data duplication, ever.** The original combined tool had the 196-entry library embedded
  *twice* (once per sub-app) inside base64-encoded blobs — that's the problem this whole rebuild
  solved. All three tools fetch the same two JSON files at runtime.
- **Stable `id` fields, decoupled from display `name`.** Renaming an entry's display name never
  breaks an alias map or verification record that points at it by id.
- **Harmonization defaults to trusting the library, logging the SFF value as reference.** This was
  an explicit policy choice (not "auto-average" or "always ask") — the reasoning: a single
  SFF-reported cost is one data point of unknown reliability, and shouldn't silently override a
  correlation fit from many data points, but it's still valuable to keep as a reference for future
  recalibration.
- **Bootstrap exponent defaults to n ≈ 0.6** (the standard six-tenths rule), explicitly editable,
  and requires a human "I've reviewed this" checkbox before it can be confirmed — a bootstrapped
  entry is always visibly provisional (`confidence: "bootstrapped"`), never silently treated as
  equivalent to a real regression fit.
- **The alias table persists across SFF files**, exported/re-imported as a plain JSON file (no
  database, no backend) — consistent with everything else in this toolkit being static files.
- **"Search the full library" is always available**, not gated behind "fuzzy match found nothing."
  Fuzzy matching can return *wrong* top candidates just as easily as *no* candidates, and a human
  should always be able to override by browsing directly.
- **Testing was done with real headless-browser runs (Playwright), not just code review**, and
  this caught real bugs worth knowing about so they don't get reintroduced:
  - A checkbox's `change` handler triggered a full panel re-render before the form fields it
    controlled had been synced into state — this silently wiped whatever the user had just typed
    into the bootstrap/manual forms. Fixed by syncing all fields into state *before* any
    re-render, in every handler, not just the "confirm" button.
  - The cost-comparison math in the SFF CapEx Matcher was missing the material-factor multiplier that
    the Estimator's own engine includes by default — found by cross-checking against an
    independent calculation, not by inspection.
  - `design_results` fields can be literature-style wrapped (with source/confidence), not just
    flat numbers — a real unit in the MSW example file (`Steam Turbine.design_results.
    electricity_production`) was being silently dropped before this was found and fixed.
  - **(SFF OpEx Matcher build)** `beginReview()` originally reset `uiState = {}` *after*
    `initSuggestion()` had already populated it for every stream — every suggested category and
    price was silently discarded before the reviewer ever saw them, and every dashboard total
    came out $0. A Node-based dry run against both example files (not just eyeballing the code)
    caught this immediately: every row printed a suggested source string but an empty price.
    Fixed by resetting state *before* parsing/suggesting, not after.
  - The dashboard's net-total color styling did `netEl.parentElement.querySelector('.val')` to
    find... itself, one step removed, which throws if `parentElement` isn't what you assume.
    Simplified to just style `netEl` directly. Caught by the same dry run, before it ever reached
    a real browser.
  - Real SFF files mix at least three different shapes for a "value with units": a bare number, a
    literature-extraction wrapper (`{value: {value, units}, source, confidence}`), and a flatter
    `{value, units}` pair with no source/confidence (seen on `stream.price` and
    `stream_properties` fields in SuperPro-simulation-style exports). The SFF CapEx Matcher's own
    `extractSpecValue` only handles the first two — it would silently drop the `units` sibling on
    the third shape. `streams_matcher.html` uses a corrected version (`extractValueUnit`) that
    checks for a sibling `units` key too. Worth porting back into `sff_matcher.html` if a future
    SFF file exposes `design_input_specs`/`design_results` in that third shape.
  - **(User bug report round, six fixes at once)** All six had the same shape of root cause —
    state that was computed for *display* but never written back to the single source of truth,
    or a re-render triggered on every keystroke rather than on a discrete action:
    - **Typing "9.5" only registered one character, and lost the decimal point.** Several inputs
      (`sff_matcher.html`'s search box and unit-converter value field; `estimator.html`'s
      qty/baseSize table cells) called a full panel/table re-render on every `input` event. That
      destroys and recreates the very `<input>` the user is typing into (losing focus after one
      keystroke) and redraws its `value` from `parseFloat()`-round-tripped state (silently
      dropping a trailing "."). Fixed by patching only the specific dependent DOM (a result span,
      a handful of per-row cost cells) instead of the whole panel/table, and never touching the
      focused input itself. `estimator.html`'s table cells had a partial workaround already (a
      `requestAnimationFrame` refocus hack) that fixed focus but not the decimal-truncation —
      removed in favor of the same targeted-patch approach.
    - **The converter's "Use this value" button silently did nothing.** `renderConverterWidget()`
      computed the category/from/to units to *display* as local fallback variables (defaulting
      unrecognized state to a sensible guess) but never wrote the resolved values back into
      `st.convCat`/`convFrom`/`convTo`. The button's click handler read those state fields
      directly — so it could see `undefined` even while the dropdowns visibly showed a valid
      selection, and `convertUnit(undefined, ...)` threw, caught by nothing. Fixed by persisting
      the resolved values back into state at the end of every render. Present in both
      `sff_matcher.html` and `streams_matcher.html` (copied architecture, copied bug) — fixed in
      both, plus wrapped both button handlers in try/catch so a future instance of this class of
      bug fails as a visible toast instead of a silent no-op.
    - **A confirmed match's size defaulted to the library entry's minimum in the Estimator.**
      `st.matchSize` was only ever set by clicking "Compute comparison" or the (then-broken) "Use
      this value" button — confirming a match without doing either sent `size: null`, and
      `estimator.html`'s `addEquipmentFromSelector()` silently clamped any null *or merely
      out-of-range* size down to `size_min`. Fixed on both ends: `sff_matcher.html` now auto-fills
      a sensible default size as soon as a match is picked (from the same guess the converter
      widget already computes) and captures whatever's typed in the size field live; and
      `estimator.html` no longer clamps a valid-but-out-of-range size at all — it now reaches the
      row and gets the same out-of-range/parallel-units treatment already used for manually-added
      rows, instead of being silently forced to the smallest possible size.
    - **Bootstrap's valid range was forced to a single point** (`smin: size, smax: size`),
      making the new entry unusable for any size other than the exact one bootstrapped. Added
      editable min/max fields defaulting to a rough ±50% bracket, clearly labeled as a guess.
    - **The cost comparison always used the base material's factor (1×), never the actual
      material.** There was no way to change material during matching at all — the comparison
      silently used `entry.mf` (always the base default) regardless of what the SFF reported for
      that unit. Added a material dropdown to the matched panel (reusing the Estimator's own
      "Pricing Materials" note-parsing so both tools agree on the options), defaulting to the
      unit's own reported material when it matches an option; the comparison now uses whichever
      material is selected, and displays the material/installed-cost factors it applied
      so this is visibly auditable rather than a black box. The selected material is now also
      passed through to the Estimator on confirm, so the row starts correct instead of always
      defaulting to base material.
- **One SFF unit can need several library entries** (a distillation column is really a vessel
  plus trays/packing plus a reboiler and condenser, but PISCES reports it as one unit). Rather
  than bolt this on as a separate "sub-component" mode, every confirm action just commits one
  component to a `components[]` array on that unit's decision and offers a second button to stay
  on the same unit and add another, instead of moving on — so the common single-component case is
  unchanged (still one click), and the multi-component case is a deliberate second click, not a
  new workflow to learn. This is also why the alias map's schema had to become "one `unit_type` →
  an array of remembered components" instead of one-to-one — a distillation column's vessel and
  its two heat exchangers are three different remembered mappings under the same `unit_type`, and
  all three should resurface as one-click shortcuts the next time that unit type appears.
- **Connected-stream context was missing from the matching decision entirely.** The review panel
  showed a unit's own specs and cost, but not what was actually flowing into or out of it — yet
  a reviewer judging whether a "Distillation Column" candidate list is even plausible needs to see
  the feed's mass flow and temperature first. Added a "Connected streams" card (joined from the
  SFF's own `streams[]` by `source_unit_id`/`sink_unit_id`), shown first, before specs — using the
  same three-shape value parsing (`extractValueUnit`) already built for `streams_matcher.html`,
  since `stream_properties` fields exhibit the exact same flat-`{value,units}` shape that
  `extractSpecValue` doesn't handle (see the bullet above about that shape).
- **(Proactive bug-hunt pass, no user report needed)** Auditing all three tools for correctness
  turned up several real issues, one of which involved a genuine mistake worth recording plainly
  so it isn't repeated:
  - **Bare-module factor (`bm`) and installation factor (`inst`) are ALTERNATIVES, not both
    applied.** A bare-module factor already bakes in installation cost (that's what "bare module"
    means in Guthrie/Seider costing) — multiplying both together double-counts it. The correct
    rule, and the one `selector.html` had implemented from the start: use `bm` when it's a real
    value (not just the `=1` placeholder many rows carry for entries where it doesn't apply),
    otherwise fall back to `inst`. **`estimator.html` and `sff_matcher.html` were multiplying both
    together unconditionally** — and on discovering the inconsistency, the first instinct was to
    treat the Estimator's formula as authoritative (it's the tool that produces the final CAPEX
    numbers) and change `selector.html` to match it. **That was backwards, and shipped briefly
    before being caught and corrected.** 118 of the library's 196 entries have both factors >1
    simultaneously (e.g. `steam-boiler-seider`: bm=2.19, inst=1.53), so the wrong direction of this
    fix would have overstated installed cost by ~53% for the majority of the library. Fixed by
    reverting `selector.html` and bringing `estimator.html`/`sff_matcher.html` to match it instead,
    via a shared `installedCostFactor()`/`installedFactor()` helper in each file (can't literally
    share code across separate static HTML files, so the same logic is duplicated three times —
    if this formula ever needs to change again, it needs to change in all three places). The
    lesson: when two tools disagree on a formula, "which one already has tests/is more central"
    is not the same question as "which one is textbook-correct" — check the source methodology
    (or ask), don't just pick the more-established-looking answer.
  - **Two real data gaps in the equipment library**, found by scanning every numeric field across
    all 196 entries for `null`: `pump-centrifugal.cepci` and `hydroclone.bm`. Both were resolved
    by comparing against sibling entries in the same category/source (every other entry in
    `pump-centrifugal`'s category uses cepci 567; every other `bm=null`-adjacent entry in
    `hydroclone`'s category with the same `inst` value uses bm=1) and fixed in
    `equipment_library.xlsx` — the master copy — then regenerated `equipment_library.json` through
    `build_equipment_library_json.py`, never hand-edited. Neither of these had a code-level guard
    before this pass: `eq.cepci` was divided into directly with no fallback in `estimator.html`/
    `sff_matcher.html` (a null value would have produced `Infinity` for that entry everywhere —
    row display, pie chart, CSV, HTML report), unlike `selector.html`, which already had a
    `|| 567` guard. Added matching fallbacks (`|| 567`, `|| 1`) in the other two tools so a future
    data-entry gap degrades gracefully instead of silently breaking.
  - **`selector.html` was discarding a legitimate but out-of-range size before ever sending it to
    the Estimator** (`addSingleLibToEstimator`, `addFamilyMemberToEstimator`) — the exact same
    "silently defaults to minimum" bug already fixed once in the SFF CapEx Matcher → Estimator handoff,
    reintroduced through a different code path that pre-filtered the size before the Estimator's
    own (already-correct) out-of-range handling ever got a chance to run. This is worth
    remembering as a *pattern*: fixing a bug in one call site doesn't fix it in every call site
    that independently reimplements the same discard-if-out-of-range logic.
  - **`transformLibraryEntry()` in `estimator.html` silently dropped the `id` field** when
    converting a raw library entry into the shape the running app uses. This broke
    `addEquipmentFromSelector()`'s "does this bootstrapped/custom entry already exist?" dedup
    check (`EQUIP_LIBRARY.find(e => e.id === item.entry.id)`) — since no transformed entry had an
    `id` at all, the check could never match anything, so confirming the same bootstrapped/manual
    entry a second time (a duplicate broadcast, a re-confirmed match, etc.) would have silently
    created a second, separate library entry instead of reusing the first. Fixed by preserving
    `id` through the transform; confirmed with a test that simulates the same confirm arriving
    twice and checks exactly one library entry (with two rows) results, not two entries.
- **(User-reported) The "size to evaluate at" wasn't reliably in the library entry's own unit.**
  The unit converter's auto-guess logic treated the SFF spec's own detected unit category as
  master, and only used the library entry's detected unit as a same-category nicety — so whenever
  a unit's first-listed spec was in a *different* physical category than the matched entry's size
  basis (e.g. the spec offers a length but the entry's correlation is sized on area), the
  converter silently defaulted to the spec's own arbitrary unit while the field right next to it
  still read "in \<entry's unit\>". The number shown (and auto-sent to the Estimator) was in the
  wrong unit entirely, with no visible sign anything was off. Fixed by flipping which side is
  master: the library entry's own detected unit now anchors the category, and the "which SFF
  spec?" dropdown now defaults to whichever spec is dimensionally *compatible* with that unit,
  not just the first one listed. Two things had to be fixed alongside this for it to actually
  work end-to-end:
  - **Switching candidates didn't reset the converter.** Picking a different suggested match
    left the previous entry's auto-guessed category/unit/spec/size in place, so the "default" the
    reviewer saw was anchored to whichever entry they'd looked at *first*, not the one currently
    selected. Fixed by resetting all of the converter/size/material state when `selectedLibId`
    changes.
  - **Composite (non-single-unit) size bases were being misdetected.** Eight entries use a
    `"S = (...)*(...)^n"` formula as their size basis (e.g. `pump-centrifugal`'s
    `"S = (Flow rate, gal/min)*(Pump head, ft)^0.5"`) — not a physical unit a converter can
    bridge to at all, since it's the product of two different quantities. The substring-based
    unit detector would still "succeed" on a fragment inside one of these (spotting "gal/min"
    inside the pump formula) and treat the whole composite as if it were that simple unit. Added
    an explicit guard (`isCompositeUnitBasis()`, keyed on the `"S = "` prefix this library
    consistently uses for composite bases) so these are always treated as non-auto-detectable —
    the widget now falls back to the spec's own category and shows a visible warning that the
    size needs a manual sanity-check, rather than silently trusting a wrong number. Confirmed with
    a test that verifies the naive detector *would* have matched "gal/min" without the guard.
  - **(Utility Duty & Annual OpEx build)** A Node dry-run harness (see §8) initially reported "0
    utility-duty entries" for every file, even though the totals it printed right below that were
    clearly nonzero. Not a bug in the tool — a bug in the *test harness's* exposure of internal
    state: `utilityDuties` was returned as a bare value (`return {utilityDuties, ...}`), which
    captures whatever it was bound to *at the moment the script finishes its top-level execution*
    (an empty array), not whatever it gets reassigned to later when `beginReview()` actually runs.
    Functions like `computeUtilityDutyTotals()` still saw the right data because they close over
    the *variable*, not a copied value. Fixed by exposing a getter function
    (`() => utilityDuties`) instead of the variable itself. Worth remembering for any future
    Node-dry-run harness against this codebase: expose state via getters, not bare references.
- **Maintenance and Labor are modeled as a flat % of Total Installed Cost, not computed from
  headcount or the equipment library's own `labor` field.** That `labor` field (Seider/Sinnott-style,
  present on 151/196 library entries) is *installation* labor — it's already folded into each
  entry's `inst`/bare-module cost multiplier, not a separate operating-cost input, and was
  confirmed as such rather than repurposed. Defaults (4% maintenance, 15% labor, both /yr of TIC)
  come from Peters & Timmerhaus / Seider / Turton's published ranges (2–10% and 10–20%
  respectively) — both fields are plainly editable, and the labor default is flagged in the UI as
  the rougher of the two approximations, since real operating labor scales with operator-headcount
  and shift structure, not capital.
- **Per-unit utility duty prices are resolved against the file's own `utilities.*` entries before
  ever touching the toolkit-default table**, exactly mirroring the boundary-stream side's
  tiered-suggestion philosophy from §5.4 — a toolkit default is a last resort, always visibly
  tagged, never silently indistinguishable from a real SFF-reported price.
- **The Estimator → Streams/OpEx Total-Installed-Cost sync reuses the CEPCI-sync pattern exactly**
  (`request-*` ping, `*-update` push on every edit and on load, relayed by the wrapper) rather than
  inventing a new cross-tab mechanism — see §4. Consistency here means a future third consumer of
  the running CAPEX total wouldn't need a third bespoke bridge, just the same two message names
  with a new pair of endpoints.

## 7. Not yet built — roadmap

> **This list is out of date as of Round 7.** Items 3, 4, 5, 6, 8 and 10 were built in Rounds
> 3–7 and are marked below. For a current, evidence-checked view of what is still missing —
> measured against the project's own stated goal rather than against this list — see
> **`claude/TEA_GAP_ASSESSMENT.md`** in the project, and the OPEN GAPS block at the top of
> `HANDOVER.md`.
>
> Of the three findings that used to outrank everything here, **two are now closed**: the
> chemical price library was built in Round 6d, and OpEx, revenue and product output all scale
> with a plant-capacity driver as of Round 7, so the cost-vs-capacity surrogate is computable and
> exportable. **What still outranks everything else is `F_P = 1`** on all 196 library rows, which
> needs Turton Appendix A and nothing else.

Roughly in order of how directly each serves the original project goal (a standardized TEA/LCA
reporting template, feeding a supply-chain/superstructure optimization model):

1. **The actual standardized report/template output.** Everything built so far produces the
   *inputs* to that (CAPEX numbers, matched equipment, verification trail, stream-level and
   per-unit-duty OpEx pricing, Annual OpEx) but there isn't yet a single "generate the
   VIFF-standard report" export that pulls it all into the template format Peter's original
   project scoped.
2. **~~Streams / OpEx matching~~ — built** (`streams_matcher.html`, §5.4), **~~per-unit utility
   duty pricing~~ — built** (§5.5). What's still missing within this scope:
   - **No `stream_alias_map.json` / `utility_alias_map.json` carried forward across sessions.**
     The SFF CapEx Matcher persists confirmed `unit_type → library id` mappings across files; neither
     OpEx mode has an equivalent yet, so a stream or duty matched to a utility this session isn't
     remembered next session. Would follow the same export/re-upload pattern if added.
   - **Fuzzy utility matching is approximate by design** (best-effort, human-reviewed, per the
     scoping decision behind this build) — it can suggest a plausible-looking but wrong match
     (e.g. an unrelated flue-gas stream scoring against a "Natural Gas" utility purely on token
     overlap, or — a real example from the MSW file — a syngas compressor's power duty scoring
     against "Natural Gas" instead of "Electricity" purely on token overlap) when there's no
     mass/energy flow to sanity-check against. Never auto-applied without the reviewer clicking
     it, but worth knowing the failure mode isn't "finds nothing," it's "finds something
     plausible-but-wrong" — same trade-off already accepted for equipment fuzzy matching in §5.3.
   - **Duty rows with no structured unit in the SFF are correctly left unpriced, not guessed at**
     — the MSW example's own furnace/compressor duty fields (e.g. `"Tar reformer furnace duty":
     {value: 6.1, ...}`) have no `units` key at all in the JSON, only a unit mentioned inside a
     text snippet (`"...Gcal h⁻¹: 6.1"`). This is a real SFF data-quality gap, not a tool bug — see
     Peter's own note in `Template_process.docx` ("Utility fields in many SFF files are left
     blank... utility demand is essential for accurate OpEx calculation"). Nothing to fix here
     without the source file itself carrying a structured unit; flagging it for the reviewer is
     the correct behavior, not a placeholder for a future fix.
3. **No profitability / discounted-cash-flow layer (NPV, IRR, payback period, minimum selling
   price).** Everything built so far produces a CAPEX total and an Annual OpEx total — genuinely
   enough for the supply-chain/superstructure optimization model's surrogate cost function, but not
   enough to reproduce the headline numbers a published TEA paper usually reports. Would need: a
   depreciation schedule (e.g. MACRS), tax rate, discount rate, plant life and construction
   schedule, working capital (and its end-of-life recovery), and — for MSP specifically — an
   iterative solve for the product price that zeroes NPV. A meaningfully large scope addition, not
   a quick one.
   - **Cheap partial win, worth doing before the full DCF engine:** SuperPro-style SFF exports
     already carry `metadata.NPV_usd`, `IRR_after_taxes_pct`, `IRR_before_taxes_pct`, `ROI_pct`,
     `payback_period_years`, `working_capital_usd`, and `total_investment_usd` as native fields
     (confirmed present in the Aspirin example) — these are the *source software's own* computed
     values, not something this toolkit would be deriving. Surfacing them in the same
     "SFF-reported totals" card the Annual OpEx summary already shows (§5.5) would be a small
     addition, not a new calculation engine, and would cover every file whose source software
     already reports them (which the MSW-style literature-extraction files don't).
4. **No indirect/whole-plant capital cost layer.** The equipment library's `bm`/`inst` factors only
   capture bare-module/installed cost *per piece of equipment* — real published capital estimates
   also add piping, instrumentation, buildings, yard improvements, service facilities, land,
   engineering, contractor's fee, contingency, and start-up costs on top (the classic
   Percentage-of-Delivered-Equipment-Cost / Lang-factor method), often 30–100%+ on top of the
   summed installed-equipment cost. Nothing in the toolkit currently adds this layer, so the
   Estimator's "Total Installed Cost" is closer to a Fixed Capital Investment *floor* than the full
   FCI, let alone Total Capital Investment (which also adds working capital).
5. **No consumables (catalysts, solvents, membranes, filter media) tracking.** Flagged in Peter's
   own `Template_process.docx` notes as "likely from self-inputs" — neither example SFF reports
   these, and there's no manual-entry mechanism for them anywhere in the toolkit yet. Would
   probably want a simple manual-line-item table in the Annual OpEx panel (§5.5), similar in
   spirit to the Estimator's "Custom Equipment" tab.
6. **~~LCA / emissions~~ — started (Round 7).** No longer out of scope. A seventh tab,
   **Life-Cycle Impacts** (`lca.html`), characterises the reviewed boundary-stream and utility
   slate against `impact_factors.json` (TRACI 2.1 categories, same provenance rules as the price
   files). It consumes one `inventory-update` message from the SFF OpEx Matcher and holds no SFF
   parser and no second copy of the inventory, so the TEA and the LCA are guaranteed to describe
   the same plant at the same scale — which is the whole argument for one tool rather than two.

   **Read the limits before quoting a number off it.** It reports a **gate-to-gate** inventory:
   direct emissions at the boundary plus purchased utilities. It is *not* cradle-to-gate, because
   `impact_factors.json` ships no cradle-to-gate data (the comprehensive LCI sets are licensed and
   the free ones are partial), and the Coverage panel states in mass terms how much of the
   boundary goes uncounted as a result. Nine of the ten TRACI categories currently have no
   factors at all and their totals are **blank, not zero** — the same policy as the empty
   pressure-factor table. See `claude/LCA_HOWTO.md` for how to fill them without introducing a
   defect.
7. **Live PISCES connection.** SFF files are uploaded by hand right now; fetching directly from
   PISCES was explicitly deferred from the start of this project.
8. **~~CEPCI auto-lookup by year~~ — built (Round 6).** The Current CEPCI Index cell carries a
   year dropdown driven from `costing_basis.json`'s `index_by_year`; see §5.2. **Round 6b** added
   the other half: `metadata.TEA_year` is now read from the SFF so an SFF-*reported* cost can be
   escalated from the year the file states, instead of sitting in a different dollar year from
   every library row beside it. (The Annual OpEx panel's utility-price
   defaults have the same "refresh periodically" characteristic — see §5.5 — and could share
   whatever periodic-refresh mechanism eventually gets built for CEPCI.)
9. **Two-point re-fitting for bootstrapped entries.** Once the same equipment type has been
   bootstrapped from more than one SFF file, the tool could detect the accumulated (size, cost)
   pairs and offer to fit a real regression, upgrading the entry from `"bootstrapped"` to
   `"regression"` confidence — this was the original intent behind keeping bootstrapped entries
   visibly provisional rather than silently equal to real library entries.
10. **~~"Use SFF cost" doesn't change the Estimator's number~~ — built.** An SFF-reported cost is
    now passed through as the row's installed cost (`fixed_cost` → `fixed_installed_cost`), tagged
    `sff_override`, and optionally escalated from `metadata.TEA_year` (Round 6b, §5.2).
    *Original note, kept for context: when a reviewer picked "use SFF-reported cost for this
    instance" during harmonization, the choice was logged in the verification record but the
    Estimator still computed the row's cost from the library formula — there was no "override
    with this literal dollar figure" slot in its row model.*
11. **A script to append `new_library_entries.json` into `equipment_library.xlsx` automatically**,
    closing the loop instead of requiring a manual paste (mirroring
    `build_equipment_library_json.py`, but in the other direction).
12. **Batch SFF processing** — one file at a time right now; if VIFF accumulates many flowsheets, a
    "process a folder of SFF files, review once, apply everywhere" mode would help.
13. **Merging two independently-reviewed verification records/alias maps** — if two people review
    different SFF files (or the same one) separately, there's no tooling yet to reconcile their
    exports into one alias map.
14. **Removing a component from the SFF CapEx Matcher doesn't retract its row from the Estimator.**
    Clicking "Remove" on a recorded component only takes it off the SFF CapEx Matcher's own list — it
    was already sent to the Estimator as an independent row the moment it was confirmed, and the
    two tools don't share a row id to reach back and delete it there too. The UI says this plainly
    next to the Remove button, but a cross-tab "retract" message (mirroring the existing
    `add-to-estimator` postMessage) would close this gap if it becomes a real papercut in practice.
15. **No equivalent of the equipment-library's "search full library" for choosing which streams
    feed which unit.** The connected-streams card (§5.3) is read-only context — useful for judging
    a match, but there's no way to correct it if the SFF's own `source_unit_id`/`sink_unit_id`
    linkage is wrong or incomplete for a given file.

## 9. The academic-methodology integration (2026-09-10)

Prompted by a comparison of the toolkit against how Turton, Seider and Towler & Sinnott actually
build a capital and operating cost estimate. The full gap analysis lives in
`TEA_Methodology_Gap_Analysis.md`; this section records only what changed in the code and why, so a
future session doesn't have to re-derive it.

### 9.1 The finding that drove the design

**The toolkit's factor placement is Seider-consistent, and Turton's is not.** The Estimator computes
`C_P × F_M × F_BM`, which is what Seider/Guthrie does. Turton computes `C_P0 × (b1 + b2 × F_M × F_P)`
— the material factor multiplies **only** the `b2` term, because `b1` represents installation costs
that don't scale with material of construction.

Nothing was wrong: the library contains no Turton rows. But the first Turton correlation anyone added
would have been silently mis-costed. Worked example, stainless shell-and-tube, `F_M` = 2.75:

| | Multiple of C_P |
|---|---|
| Turton, correct: `1.63 + 1.66 × 2.75` | **6.195** |
| Seider-shaped: `2.75 × 3.29` | **9.0475** |

A 46% overstatement. Both branches are now implemented and pinned by
`tests/test_pressure_and_conventions.js` §1e. **The lesson: fix the schema before the data arrives,
not after.**

### 9.2 What was added, by phase

- **Phase 0 — schema.** 13 columns added to `equipment_library.xlsx` (`factor_convention`, `b1`,
  `b2`, `fp_model`, `fp_c1..c3`, `p_basis_barg`, `fp_review`, `equip_class`, `cost_is_installed`,
  `factor_review`, `basis_year`). All are **optional** in `build_equipment_library_json.py` and all
  default to values reproducing the pre-Phase-0 arithmetic exactly. The builder now validates the
  enums and hard-fails a `turton` row missing `b1`/`b2`, rather than letting it fall through to the
  Seider branch as a silently wrong dollar figure.
- **Phase 1 — pressure factor.** Turton's vessel wall-thickness formula plus a log-quadratic model, in
  `estimator.html`. **Every shipped row has `fp_model = 'none'`, so F_P = 1 and Phase 1 is
  arithmetically inert until a human opts a row in** — verified by a test that feeds 50 barg to all
  196 rows and asserts none of them move.
- **Design pressure from the SFF.** `sff_matcher.html` suggests `max(connected stream P) × 1.1`, shows
  the arithmetic and the stream it came from, and lets the reviewer override. This was the cheap part:
  the connected-streams card was *already* parsing and displaying stream pressures.
- **Σ C_BM reported separately from "Total Installed Cost."** Every downstream factor in all three
  references keys off the bare-module sum, and Turton's grass-roots term needs the base-condition sum
  too. Both are accumulated and broadcast.
- **Phases 2 / 3 / 3.5 / 4 / 5 — `plant_economics.html`.** See §5.6.

### 9.3 Design decisions worth not re-litigating

- **Never pick a convention for the user.** All three cascades are computed from the same `Σ C_BM` and
  shown side by side with the spread as a headline figure. On a 10M bare-module sum the three standard
  textbooks disagree by **23%** on FCI. For a *harmonisation* project, a tool that hides that
  disagreement behind one chosen number is worse than one that shows it. The spread is an output, not
  a defect.
- **Factors live in `costing_basis.json`, not in the HTML.** Same reasoning as the equipment library:
  a collaborator retuning an assumption shouldn't have to open a `.html` file.
- **Every factor shows its published range and is editable.** Out-of-range values are flagged, not
  blocked — the reviewer may well be right.
- **Turton's F_P log-quadratic coefficients are deliberately NOT shipped.** Transcribing Appendix A
  tables from memory into the toolkit's data layer would put unverified numbers into a source of
  truth. `fp_review` marks the 59 rows that want a model; `costing_basis.json` holds a labelled empty
  placeholder to fill in from the book.
- **Labour uses Turton's correlation regardless of which capital convention is selected**, because it
  is the only headcount model of the three evaluable off a flowsheet. It replaces the old "labour =
  15% of TIC", which gave a high-pressure low-headcount plant and a solids-handling high-headcount
  plant identical labour at equal capital. `N_np` and `P` are counted from `equip_class` and are
  **editable**, because:
  - **The library has no column-shell entries.** A tower is a Pressure Vessel row plus a Tray/Packing
    row, both counting as zero toward `N_np` even though Turton counts the tower as one step. The UI
    flags this when vessel/internals rows are present rather than auto-promoting them — a vessel is
    sometimes genuinely just a knockout drum.
- **Seider's working capital is implicit.** It is 15% *of TCI*, and TCI contains it, so
  `C_TCI = C_TPI / (1 − f)`. The naive `C_TPI × (1 + f)` understates TCI by 2.3%. Pinned by test.
- **SFF-reported totals are read-only and never enter a sum** — the same policy §5.5 already applies
  to `annual_operating_cost_usd`. The card cross-checks and flags gaps above 30%.
- **A missing input leaves a figure blank and flagged; it never becomes zero.** Zero is a claim, blank
  is an absence, and conflating them is how a factored estimate quietly loses a third of its capital.

### 9.4 Bugs and data issues this work surfaced

- **`suggestDesignPressure()` called a function that doesn't exist** (`detectUnit` instead of
  `detectUnitCategory`) — a `ReferenceError` that would have thrown on every unit review. Caught by
  the end-to-end Node dry run, not by reading the code. *That is the third time in this project's
  history a dry run has caught something inspection missed.*
- **`detectUnitCategory()` had no kPa/Pa/MPa branch at all.** SFF `stream_properties.pressure`
  commonly reports kPa, so the design-pressure suggestion would have said "unit not recognised" for
  precisely the files most likely to carry a usable pressure. Added, ordered before the bare-`bar`
  test so `mbar` can't shadow them.
- **4 library rows have a `source` that disagrees with their `cepci` basis.** They agree with their
  *category* on three independent fields — category name, CEPCI basis, and size unit (Seider quotes
  ft², Sinnott m²) — and disagree only on `source`. Three fields against one means the **source
  string** is wrong and **no cost is affected**; it is a citation error. The four look cross-swapped
  between the two heat-exchanger categories: `heat-exchanger-air-cooled-fin-fan`,
  `heat-exchanger-plate-and-frame-suggested`, `heat-exchanger-double-pipe`,
  `heat-exchanger-floating-head-shell-and-tube`. **Not auto-corrected** — the build script warns and a
  human decides, same policy as `fp_review`. Recorded as a bounded `KNOWN_SOURCE_MISTAGS` list in the
  test so it stays visible without leaving a permanently-red suite.
- **"Base conditions" is not synonymous with carbon steel.** 48 of 196 rows are fitted on something
  else (19 stainless, 8 on 304 SS, 8 cast iron, …), and 23 list a material option *cheaper* than their
  own base — carbon steel at 0.5× on a stainless-based exchanger correlation, for instance. So the
  base-condition bare-module sum can legitimately come out **above** the as-built sum. A first draft
  of the test asserted otherwise and failed against correct code. The invariant that actually holds,
  and is now tested, is `C_BM / C_BM(base) == F_M × F_P` per row.
- **CEPCI is paywalled.** Effective September 2024 it is no longer printed in the magazine and needs a
  subscription, so the year→index table in `costing_basis.json` is manually curated with an explicit
  `unverified_after` marker; anything past 2024 in it is a placeholder. Cost professionals also advise
  against escalating more than ~5 years, which means escalating a 2010 Towler–Sinnott correlation to
  today is already outside the recommended window — the tool says so rather than hiding it.

### 9.5 Still open

- Populate Turton Appendix A's log-quadratic coefficients for the 59 `fp_review` rows. Since
  Round 6c the place to put them is `costing_basis.json → pressure_factor.logquad_coefficients`,
  keyed by `equip_class` (one entry per class) or by library id; a row's own `fp_c1..c3` still
  override, and that block is now actually read — before Round 6c it was not. Then set those rows'
  `fp_model` to `logquad` in the **xlsx** and regenerate. Do **not** extend this to the shell-mass
  `Pressure vessel` rows: shell mass already embeds wall thickness, so it would double-count.
- Decide the 14 `factor_review` rows: data hole, or `cost_is_installed = TRUE`? For skid-mounted
  membrane units the latter is plausible; verify against Seider before changing anything.
- Fix the 4 `source` strings above.
- Verify Seider's cost-sheet percentages against the edition being cited — editions differ in the
  second decimal, and the values in `costing_basis.json` are flagged accordingly.
- Validate the DCF against the Aspirin file's own reported NPV/IRR using *real* equipment matches
  (the end-to-end test uses deliberately arbitrary matches, so its 22% cross-check gap means nothing).
- ~~**For the surrogate cost function: fit on `C_BM`, not on TCI.**~~ **Built in Round 7**, exactly
  as specified here: capital fitted on the bare-module sum so the optimisation model can apply the
  cascade analytically and vary contingency, OSBL fraction or working capital without refitting;
  variable cost fitted per unit capacity through the origin so the capital-fraction fixed costs
  stay analytic. See §5.8 and `claude/PLANT_SCALE.md`.
- **The MSP solver does not re-solve the cost of manufacture against the trial price.** It re-runs
  the cash flow at each trial price but holds `comCash` fixed, so the revenue-keyed COM lines
  (SG&A, R&D, marketing) sit at the entered revenue rather than the trial price's. Pre-existing
  single-point behaviour; Round 7 made it visible at every point of a sweep. Zero effect on
  Turton, second-order on Seider and Towler–Sinnott.

## 8. If you're a new Claude session starting from this doc

- Read §6 **and §9.3** before proposing an architecture change already decided against.
- The four original tools were left alone by the 2026-09-10 update wherever possible. The new fifth
  tab is a pure consumer: it reads `costs-update` and `opex-update` and never writes back.
- The equipment library is 196 entries; don't try to re-derive it from anything other than
  `equipment_library.xlsx` → `build_equipment_library_json.py`.
- If asked to edit any of the four tools, check whether `tests/*.js` already covers the area
  you're touching, and re-run those after — several real bugs were only caught this way, not by
  reading the code. Before trusting any numeric output from a new tool, do a dry run against both
  example SFF files and hand-check a couple of line items against the source paper's own reported
  figures if possible (this is how the streams-matcher unit-conversion bugs were caught — see §6).
- The two SFF example files (`Municipal_Solid_Waste_-_Methanol_via_Gasification.txt` /
  `Version_B_Acetic_Anhydride__Salicylic_Acid___Aspirin.txt`) are genuinely different schema
  styles (literature-extraction vs. simulation-export) — always sanity-check any SFF-parsing
  change against both, not just one. Both files are also copied into `site/` alongside the tools
  so `tests/*.js` can upload them directly without any extra setup.
- **If your sandbox can't reach Playwright's browser-binary CDN** (network egress restricted to an
  allowlist that doesn't include it — this happened during the utility-duty/Annual-OpEx build),
  don't skip testing, do a Node dry run instead: load the tool's `<script>` block into a `new
  Function('document','window', scriptSrc + 'return {...}')` with a minimal stubbed `document`/
  `window` (a `Proxy` that no-ops on `addEventListener`/`classList`/`style`/etc. is enough), call
  the parsing/computation functions directly against the real example SFF files, and print the
  results. This caught the real logic bugs described in §6 just as well as a browser would have —
  the only thing it can't verify is the UI actually rendering. **Expose internal state via getter
  functions, not bare variable references** (see the §6 entry on this) — a bare reference captures
  a stale snapshot from before any of the tool's own functions (like `beginReview()`) get called.
