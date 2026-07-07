# PAL Lighting Tech Support Guide — Project Context

## What this is
A single-file HTML troubleshooting guide (`index.html`) used by Roots CX's support team
to diagnose and resolve PAL Lighting customer calls. Hosted on GitHub Pages.
Images live in `assets/` (extracted from base64 in July 2026 to cut file size from
4.38MB to ~95KB — do not re-embed images as base64 going forward; always save to
`assets/` and reference by relative path).

## Long-term vision (not in scope yet)
Eventually this becomes the seed content for a RAG-based AI system (vector DB, API layer,
HubSpot integration) — a reusable framework for Roots CX clients generally, with PAL as
the first instance. That work is deliberately deferred until this static guide is
thoroughly stress-tested against real support cases. Don't start building RAG
infrastructure unless explicitly asked.

## Data structure (index.html)
- A JS array called `DATA`, embedded in a `<script>` tag — this is both the content
  store and what the render logic reads from.
- Each entry is one card: a product, a category hub, or a framework/diagnostic section.
- Common fields: `id` (unique, used for linking/nav), `category`, `backTo` (drives the
  dynamic "← Back to X" label), plus content fields that vary somewhat by card type —
  facts/specs, troubleshooting steps, DIP switch tables, escalation notes.
- `HIDDEN_BY_DEFAULT` — a JS set controlling which cards show on the default browse view
  vs. only being reachable via search/chips/drill-down.
- Adding a new product typically means updating three places: the `DATA` array itself,
  the `productSelect` dropdown, and the relevant hub's clickable link list (plus
  `HIDDEN_BY_DEFAULT` if it should nest under a category rather than show top-level).

## Validating changes
Before committing edits to the DATA array, validate it parses:
```
node -e "
const fs = require('fs');
const content = fs.readFileSync('index.html', 'utf8');
const match = content.match(/const DATA = \[[\s\S]*?\];/);
const fn = new Function(match[0] + '; return DATA;');
const DATA = fn();
console.log('DATA parsed OK. Entries:', DATA.length);
"
```
Don't use bare `eval()` — `const` declarations don't leak into the outer scope.

## Key people
- **Cory** — owner of this project, Roots CX.
- **Jason Delgado** — Florida-based field expert (installer/troubleshooter). Vets all
  new diagnostic/structural content before it goes live, and is the escalation point
  for defect-vs-installation-issue ambiguity. Anything structurally new — especially
  diagnostic decision logic — goes to Jason before being built into the HTML.

## Working principles (don't deviate from these)
- **Preview before commit.** Never push directly to `main` without Cory reviewing first,
  unless Cory explicitly says to just commit.
- **Don't silently resolve ambiguous logic.** If a troubleshooting/decision branch has
  defensible alternatives, flag it explicitly rather than picking one.
- **Concise over verbose.** Short subtitles/box text in any diagrams; don't over-explain
  when a preview or visual is what's wanted.
- **Real cases surface gaps.** Content gaps get found via live-case stress-testing, not
  speculative completeness.
- **Keep this file current.** Whenever a content change is completed (new card, closed
  gap, updated fact, etc.), update the relevant section of this CLAUDE.md before ending
  the session — don't let it drift out of sync with `index.html`.

## Known technical facts about PAL products (don't re-derive, just use)
- All PAL light heads are LED regardless of housing style — different housings serve
  different pool applications but share the same electrical behavior. Diagnostic content
  should be a shared module, not duplicated per product.
- Cloning into competitor automation systems requires TWO matching settings: driver
  protocol (software/menu) AND DIP switch bank (physical). Mismatches between these two
  are a high-volume support call driver. Protocol mappings: Pentair → IntelliBrite,
  Jandy → LED, Hayward → ColorLogic.
- PAL driver evolution: 12V → 24V DC, two-wire → four-wire connections, two-button →
  one-button interior. Relevant for diagnosing mixed-generation field installs.
- **Recurring 12V/24V copy-paste artifact across PAL's own manuals:** confirmed
  independently in the Evenglow Nicheless, Treo Max+, Treo Retro, Treo Mini+,
  PCR-3D-120/500, and PCR-3DMX-8Z source manuals — each has a troubleshooting-table row,
  install-diagram callout, or title/notes pairing saying "12volts DC"/"12 VAC"/"12V D/C"
  that contradicts 24V/24VAC stated repeatedly elsewhere in the same document (and in Treo
  Retro's case, the driver page also mislabels the product as "the PAL Treo Max"). This is
  a stale/reused table template, not a real spec difference — treat 24V DC as correct and
  flag the discrepancy in-card rather than rewriting the source manual's wording.
- **Recurring product-name copy-paste artifacts in PAL's own manuals:** separate from the
  12V/24V pattern above — some manuals also carry mismatched product names left over from
  whatever template they were built from (Treo Retro's driver page calling itself "the PAL
  Treo Max"; PCR-3DMX-8Z's install-steps heading and DMX Operation section referring to
  itself as "PCR-2MX" / "the PCR 2Z"). Flag in-card, don't silently correct the source
  manual's wording.
- Water intrusion is the most common root cause of shorts in flashing-light scenarios.
- The "V3 PCR-1Z-SM" switch-mode driver board is physically the same PCB as the PCR-2DMX/
  PCR-4 DMX board (silkscreened "42PCR4DMX") — confirmed in the PCR-1Z/2Z/1Z-SM combined
  manual, which explicitly says to ignore all DMX terminology/switch configs (dipswitch #9
  Long Strip mode, Trigger Source jumper) when the board is running in switch mode. Keep
  that content scoped to the DMX cards (`pcr2dmx`) only — don't add it to the `drivers`
  card just because the board looks the same.
- PCR-2Z (dual zone) and V2 PCR-1Z-SM boards have a second "Other Config" DIP bank,
  separate from the cloning DIP bank, that sets zone assignment (single/dual, which zone)
  and approximate white color temperature (6000K/5000K/4000K/3000K via switches 3+4). This
  wasn't documented anywhere in the guide until the `drivers` card refresh — confirmed in
  the official PCR-1Z/2Z/1Z-SM manual.

## Open content gaps (known, not yet built)
- **PCR-5S** (5-channel relay controller) — entirely undocumented in the guide.
- **Canadian Retro**, **Evenglow Sonar Retro**, **TreoMicro Nicheless**, **Evenglow
  Fiberglass Wet Niche** — four products with source manuals on hand (`source-manuals/`)
  but no card in `DATA` yet. In progress — see "Image refresh / new-card batch" below.

Note: the LED Bubbler (Niche Bubbler) troubleshooting card now exists in `DATA`
(id: `ledbubbler`) — no longer an open gap. Aqualumin Replacement (id: `aqualumin`) is
also now built — see batch status below. Several bubbler install videos, and Treo Micro
videos, still sit unmapped in the "Other PAL Videos" catch-all card (id: `morevideos`)
rather than being attached to their own card — resolve once TreoMicro Nicheless is built.
The Sonar-branded videos in that same catch-all (PAL Sonar Remote Programming ×2, PAL
Sonar Light, PAL Canadian Sonar Light, PAL Retro Lamp/Bulb Troubleshooting, PAL Retro
Light) were **not** moved to the new `aqualumin` card — unconfirmed whether they show this
specific AquaLumin-niche product or the still-unbuilt Evenglow Sonar Retro (same Sonar
tech platform, different niche adapter). Confirm actual video content before attaching to
either card.

## Image refresh / new-card batch (in progress)
Cory is having all product images redone from official manuals rather than the original
sloppy crops. Workflow: Cory drops PDFs into `source-manuals/<ProductName>/` (gitignored,
never committed); images are extracted via PyMuPDF + Pillow at high DPI (350), cropped
tight, saved as numbered `assets/image_NN.jpg|png`, and wired into the matching `DATA`
card. Cross-manual factual conflicts (see 12V/24V note below) are flagged in-card, not
silently resolved.

Status as of 2026-07-06:
- **Evenglow** — split into two cards (`evenglow` = Niche, `evenglownicheless` =
  Nicheless) since install process differs fundamentally (lift-out-and-replace vs.
  drill-and-thread). Both refreshed. Done.
- **Treo Max+ (V2)** (`treomax`) — refreshed. Done.
- **Treo Retro** (`treoretro`) — refreshed. Done. Single card (unlike Evenglow) since one
  manual covers both niche and nicheless mounting for this product.
- **Treo Mini+ (V2)** (`treomini`) — refreshed. Done. Added a Winterization section
  (wasn't present before) including the groundwater-check/no-partial-drain-at-light-height
  notes from the source manual; fixed the card's own stale "Faulty driver: 12V DC" line to
  24V DC and flagged the source manual's matching 12VAC/24VAC install-diagram artifact.
- **Aqualumin Replacement** (`aqualumin`) — new card, built. Genuinely different product
  family from the rest of this guide: retrofits a new PAL bracket + light into an existing
  Pentair AquaLumin niche, runs on its own PCR-2T-65 transformer (not the PCR-1Z/2Z family),
  and clones to competitor automation by holding a channel button on its "Sonar" remote
  instead of a DIP switch bank — flagged prominently in-card so techs don't reach for the
  Drivers card's DIP table by habit. Also documents the one-time gel-pack kit for reusing
  an existing 2-wire cable instead of running new cable. No 12V/24V artifact found in this
  manual (unlike the other refreshed products) — it consistently states 24V DC throughout.
- **Canadian Retro**, **Evenglow Sonar Retro**, **TreoMicro Nicheless**, **Evenglow
  Fiberglass Wet Niche** — new cards, not yet built.
- No live push has gone out for any of this batch yet — holding per Cory's request until
  he says to go live.

## Drivers / Controllers refresh batch (in progress)
Separate from the Lights batch above — Cory added a further-organized
`source-manuals/Drivers and Controllers/<Product>/` layer with real manuals for the
driver/controller family. Same workflow as the Lights batch (PyMuPDF + Pillow @350dpi,
numbered `assets/image_NN.jpg`, flag discrepancies in-card). Proceeding one driver at a
time, pausing for Cory after each.

**Hero photo rule (corrected 2026-07-07):** each `Drivers and Controllers/<Product>/`
folder includes a standalone clean product JPEG (e.g. `slimherosingle.jpg`,
`medherosingle.jpg`, `medherosingleDMX.jpg`, `maxherosingle.jpg`) — always use that file
directly as the card's hero `photo`, not a crop from a sell-sheet PDF render (which
carries pool-background clutter, phone/remote props, etc.). The `pcr2dmx` card had this
right from the start; the `drivers` and `pcr3d500` cards initially used sell-sheet crops
and were corrected to use the provided JPEGs after Cory flagged it.
For a card that covers more than one physical enclosure (the `drivers` card covers both
the slim PCR-1Z/PCR-1Z-SM single-zone body and the taller PCR-2Z dual-zone body), Cory
wants both product JPEGs shown side by side rather than picking just one — the render
logic only supports a single `photo` field (see `index.html` ~line 1132), so this is done
by compositing the two source JPEGs into one image (`assets/image_65.jpg`, built with
PIL, white background, thin gray divider) rather than changing the render logic. Kept
text-label-free after an initial version with captions under each photo turned out
illegible at the card's 260px display width — the card's own facts text already states
which is single-zone vs dual-zone, so the photo doesn't need to carry that itself.
**Diagram-crop rule:** when cropping a diagram/table out of a rendered manual page, verify
the crop against the full page before wiring it in — content that's visually continuous
(connected by a callout line, or a table immediately followed by note bullets) must be
captured as one complete image, not truncated at an arbitrary pixel cutoff. Several early
crops in the `drivers` and `pcr3d500` cards cut off mid-diagram or mid-sentence (e.g. a
wire-nut splice inset, a "Note:" heading, an "Other Config Operation" bullet list) and had
to be redone wider/taller. Row/column whitespace-band detection on the source render
(rather than eyeballing a fixed pixel box) is the reliable way to find true content
bounds — but pad the bounds generously, since sparse text near an edge can fall into gaps
that a coarse band scan misses.

Status as of 2026-07-07:
- **Drivers (PCR-1Z / PCR-2Z / PCR-1Z-SM / PCR-1ZW / PCR-2ZW)** (`drivers`) — refreshed.
  Done. Sourced from the combined 18-page manual (`PCR-1Z-SM1Z2Z-2.pdf`) plus the three
  individual sell sheets. Added the previously-undocumented Zone/Color-Temperature "Other
  Config" DIP bank (PCR-2Z + V2 PCR-1Z-SM only) and a new "Automation Clone Mode —
  Dedicated Output Relay" reference. Corrected the remote-pairing steps to the manual's
  official "Matching Code" procedure (power up, press Z1 once within 5 seconds) and added
  the "Clearing Code" procedure (hold Z1 3 seconds) as a new troubleshooting option — see
  "Pending sign-off" below, this replaces a materially different procedure the card
  previously documented (Code Setting button + Z1 ×3) and needs Jason's confirmation.
  Left the existing "PCR-2Z = up to 5 lights" fact alone — the new manual doesn't confirm
  or contradict it (only states 2 output conduit knockouts per zone), no solid basis to
  change it either way.
- **PCR-2DMX Driver (2-Zone DMX)** (`pcr2dmx`) — refreshed. Done. Its "Updates
  August–September 2025" addendum turned out to be a standalone V3-board quick-reference
  handout, not a spec change — content is identical to the main manual, nothing new or
  contradictory. Replaced the old "table not reproduced here, pull the source manual"
  apology with the actual full 3-channel and 4-channel per-address DIP switch tables as
  zoomable images. Also resolved the old "cloning table was garbled by PDF extraction"
  caveat — confirmed against the clean manual scan, it matches the Drivers card's 3-switch
  table exactly.
- **PCR-3D-120/500** (`pcr3d500`) — refreshed. Done. Unlike PCR-2DMX, its "Updates
  August–September 2025" addendum is **not** a duplicate — it describes a genuinely new
  "3D and 3DMX High Powered" V3 board (SKUs 64-PCR-3D / 64-PCR-3DW / 64-PCR-3DMX,
  excluding the 8Z variant) that is mode-selected via dip switches A/B into 4 distinct
  operating modes (Cloner / Module / PCR8-legacy / DMX), each with its own dip-switch
  semantics — a real architecture change, not a copy-paste artifact. Per the "anything
  structurally new goes to Jason first" rule, this was **not** built into the card as
  diagnostic logic — the card only carries a note-box flagging that the newer board and
  its 4 modes exist and aren't yet documented here. See "Pending sign-off" below. Also
  flagged two other things found while refreshing: (1) the same 12V/24V copy-paste
  artifact confirmed in this manual too (6th instance — 24V DC is correct); (2) a
  wattage/SKU naming mismatch — this manual's own title is "PCR-3D-120/500," but PAL's
  current sell sheet lists the PCR-3D as 300W/24V only (SKU 64-PCR-3D-300 /
  64-PCR-3DW-300 for WiFi) — flagged in-card rather than silently picking one figure.
- **PCR-3DMX-8Z** (`pcr3dmx8z`) — refreshed. Done. Confirmed its "Updates August–September
  2025" addendum is the same PCR4-3D/3DMX-Updates PDF used for `pcr3d500`, and its scope
  line explicitly reads "Affected part numbers: 64-PCR-3D / 64-PCR-3DW / 64-PCR-3DMX
  (excludes the 8Z)" — so the V3 "High Powered" 4-mode board does not apply to this card;
  no Jason-review flag needed here. Replaced the old "pull the 9-switch table from the
  source PDF rather than estimating" placeholder with the actual full Starting Channel DIP
  table (drivers 1-22) and 8-Zone Channel Allocations table as a zoomable image. Found and
  flagged three things in this manual: (1) the 7th confirmed instance of the recurring
  12V/24V copy-paste artifact (title + one note section say 12V DC, "IMPORTANT
  INFORMATION" says 24V DC twice — 24V DC is correct); (2) the manual's own install-steps
  heading and DMX Operation section were copy-pasted from a 2-zone product and never
  updated ("INSTALLATION INSTRUCTIONS FOR PCR-2MX RGB OUTDOOR DRIVER" / "remote DMX
  control over the two zone outputs in the PCR 2Z") — flagged in-card so a tech cross-
  referencing the customer's own manual isn't thrown by it; (3) the manual's own Starting
  Channel Table has two "STARTING CHANNEL" columns that agree for drivers 1-4 but diverge
  by a consistent 3 channels from driver 5 onward (e.g. driver 5: 97 vs 100) — flagged
  in-card rather than silently picking one column, with a note to verify the live starting
  channel against the DMX controller.
- **Color Touch Series 2 (PCR-2D), LED Optics Series 3 (PCR-300)** — new cards, have real
  manuals, not yet built.
- **Color Touch Series 4 (PCR-4), Commander Series 2, LED Optics Series 2, Power Supply
  Series 2 (PCR-2T)** — photo only, no manual provided yet. PCR-2T is already referenced
  informally in the Aqualumin Replacement card's facts (its required transformer) — that
  reference doesn't currently link to a driver card since no PCR-2T card exists.

## Pending sign-off
Decision-tree diagrams (Master Triage, Driver Power and Manual Test, Cloning and DIP
Switch Check, White/Primary Color Test) were sent to Jason as a standalone PDF for review.
Two explicit judgment calls are flagged for him:
1. Whether the cloning check always precedes the color test.
2. Whether a failed white-mode test loops back to driver internals or goes straight to
   replacement.
**Do not build these diagrams into the HTML until Jason has signed off.**

**New (2026-07-07):** the `drivers` card's remote-pairing step was changed to match the
official PCR-1Z/2Z/1Z-SM manual's "Matching Code" procedure (apply power, press Z1 once
within 5 seconds). The guide previously said to press Code Setting then Z1 three times —
that wording is flagged in-card, not deleted, since it may reflect real behavior on an
older board revision. Jason should confirm which procedure techs should actually lead
with before this is fully resolved one way or the other.

**New (2026-07-07):** the PCR-3D's "Updates August–September 2025" addendum describes a
newer "3D and 3DMX High Powered" V3 board (SKUs 64-PCR-3D / 64-PCR-3DW / 64-PCR-3DMX,
excluding the 8Z) with 4 dip-switch-selected operating modes (Cloner / Module /
PCR8-legacy / DMX), each with different dip-switch meanings from the standard board
documented in the `pcr3d500` card today. This is structurally new diagnostic content, so
per the standing rule it has **not** been built into the HTML beyond a flag noting it
exists — Jason needs to review the 4-mode logic (mode-select semantics, and whether/how
it should merge with or replace the existing Cloning/Zone tables) before it's built out
as real card content.

## Handoff / IP considerations (background — not an active task)
Cory needs a clean contract-exit path: PAL should be able to keep updating this guide
after the engagement ends, with no ongoing dependency on Cory's accounts/infra. Current
plan: transfer this GitHub repo to PAL's own org, and PAL links/iframes it from a
HubSpot page. This repo is intentionally simple (no backend, no API keys) so that
handoff is just a repo transfer — keep it that way unless told otherwise.
