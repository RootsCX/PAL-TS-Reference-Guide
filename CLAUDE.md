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
- **No build-process talk inside `index.html` (corrected 2026-07-07).** `index.html` is a
  finished product for the support team, not a running conversation about building it.
  Never write things like "Coverage gap: only a photo was provided, flag to Cory/Jason",
  "flagging for David to confirm," or "not yet built into this guide" into card content —
  that belongs in this file, not in front of a tech on a live call. Real, in-workflow
  escalation instructions ("Escalate to Jason" for structural/defect-ambiguity calls, per
  Key People above) are fine and should stay — the distinction is whether the sentence is
  part of the actual support workflow or a note about the guide's own construction.
  Technical flags (e.g. "this manual contradicts itself, X is more reliable") also stay,
  reworded to drop any reference to a person or to CLAUDE.md/the build process.

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
- **Canadian Retro**, **TreoMicro Nicheless** — two remaining products with source manuals
  on hand (`source-manuals/`) but no card in `DATA` yet. In progress, one at a time,
  pausing for Cory after each — see "Image refresh / new-card batch" below.

Note: the LED Bubbler (Niche Bubbler) troubleshooting card now exists in `DATA`
(id: `ledbubbler`) — no longer an open gap. Aqualumin Replacement (id: `aqualumin`) and
Sonar Retro Bulb (id: `sonarretro`) are also now built — see batch status below. Several
bubbler install videos, and Treo Micro videos, still sit unmapped in the "Other PAL
Videos" catch-all card (id: `morevideos`) rather than being attached to their own card —
resolve once TreoMicro Nicheless is built. Of the Sonar-branded videos in that catch-all,
"PAL Sonar Remote Programming" (×2) and "PAL Retro Lamp/Bulb Troubleshooting" / "PAL Retro
Light" have now been moved to the `sonarretro` card (confirmed match — see batch status).
"PAL Sonar Light" and "PAL Canadian Sonar Light" remain unmapped — still unconfirmed
whether they show the `sonarretro` product or the still-unbuilt Canadian Retro (same Sonar
tech platform, different niche adapter); resolve once Canadian Retro is built.

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
- **Sonar Retro Bulb** (`sonarretro`) — new card, built (2026-07-07). This is the product
  behind CLAUDE.md's old "Evenglow Sonar Retro" gap name — Cory's `source-manuals/Pool
  Lighting/Retro Lamp/` folder turned out to contain the identical two source files as the
  older `Evenglow Sonar Retro/` folder (same byte sizes), and the manual's own title is
  "Sonar Retro Bulb" — renamed the gap to match the real product name. Genuinely different
  install mechanism from every other Lights card so far: it's a screw-in LED bulb retrofit
  into an **existing** PAR38 fixture (any niche brand) — the customer's housing/lens/gasket
  stay in place, only the bulb is swapped. No DIP switches; zone linking and OEM cloning
  (channels 1–5, including Astral) are both done via the handheld Sonar remote, same
  pairing/cloning/unpairing shape as the Aqualumin card but with one more channel (Astral)
  than Aqualumin's remote reference shows — flagged in both cards as a possible hardware
  difference, not resolved. Also flagged: the manual's own Table of Contents lists a
  "Nicheless Installation" section (page 3) and a "Troubleshooting" section (page 8) that
  don't actually exist — page 3 is real Bulb Installation steps and page 8 is just a
  contact-info footer. No numeric voltage is stated anywhere in this manual (no 12V/24V
  artifact to flag), but it does ship in genuinely separate Low Voltage and "-120" (120V
  mains) SKU families. Moved 4 videos out of the `morevideos` catch-all onto this card
  ("PAL Sonar Remote Programming" ×2, "PAL Retro Lamp/Bulb Troubleshooting", "PAL Retro
  Light") and updated the cross-reference notes on `aqualumin` and `treoretro` accordingly.
  Images: hero (`image_94.jpg`) extracted as an isolated alpha-masked cutout from the sell
  sheet's embedded image XObjects (the sell sheet's photography layers a semi-transparent
  bulb render over a pool background, so a plain page-region crop kept picking up
  background bleed-through — pulling the individual image XObject with its soft mask gave
  a clean cutout instead). Steps 1–8 and the remote diagram (`image_95.jpg`–`image_98.jpg`)
  are vector line art in the install manual, autocropped to content bounds via PIL
  `ImageChops.difference` against a white background rather than eyeballed pixel boxes.
- **Evenglow Fiberglass (Wet Niche)** (`evenglowfiberglass`) — new card, built (2026-07-07).
  Two source files: a 5-page install/troubleshooting manual (`64-EGF-Install-Guide-2.pdf`,
  "PAL-EGF") and a 2-page sell sheet (`PALWetNiche-2.pdf`). Facts/issues/steps sourced from
  the install manual, which is internally consistent at 12V DC throughout (PCR-4/PCR-2D,
  same driver family as Evenglow Niche/Nicheless) — no self-contradiction in this specific
  document. Three findings, none silently resolved: (1) **Hole-size conflict** — the install
  manual specifies a 2⅜"/2½" holesaw for the wall-mount hole, but the sell sheet specifies
  1⅞" (the same size already used on the Evenglow Nicheless card) — a real diameter
  mismatch for what should be the same physical fitting, not a typo either side can be
  confidently picked over. Flagged as an escalate-level warning since drilling the wrong
  size is irreversible — added to Pending Sign-off for Jason. (2) The sell sheet documents
  a newer 24V PCR-1Z/2Z driver generation, same "Newer Generation" pattern already flagged
  on Evenglow Niche/Nicheless — plus a notable data point for the ongoing PCR-2D voltage
  question: this sell sheet's own "Design & Features" bullet reads "Low Power / 12/24V DC"
  in one line, on the same sheet whose driver-spec box elsewhere states "24V DC" for the
  PCR-1Z/2Z — could mean the light body is driver-generation-agnostic, or could be another
  copy-paste artifact; not resolved. (3) Minor/cosmetic — the install manual's own Table of
  Contents lists page 3 as "Nicheless Installation" but the actual page 3 heading reads "Wet
  Niche Installation" (same reused-ToC-template pattern seen elsewhere, no instructional
  impact). Images: hero (`image_99.jpg`) pulled the same way as the Sonar Retro Bulb's —
  isolated alpha-masked image XObject from the sell sheet rather than a flat page crop.
  Install diagrams (`image_100.jpg`–`image_104.jpg`) are vector line art spread across
  two-page-per-PDF-page spreads; split into left/right halves and autocropped via PIL
  `ImageChops.difference` before saving. Reused the existing `image_31.jpg` (24V PCR-1Z
  wiring overview) for the Newer-Generation note-box rather than re-extracting it.
- **Canadian Retro**, **TreoMicro Nicheless** — new cards, not yet built. Proceeding one at
  a time in that order, pausing for Cory after each, per his request (2026-07-07).
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
- **PCR-2D Driver (Color Touch Series 2)** (`pcr2d`) — built. Done, but lighter than the
  cards above: the only source on file (`PALPCR-2D-24.pdf`) is a 2-page sell sheet, not an
  install/troubleshooting manual — no board diagram, DIP table, or steps exist to include,
  flagged in-card as a coverage gap rather than fabricated. Two findings: (1) the sell
  sheet's own product photography (used as this card's hero image) shows the enclosure
  sticker printed "OUTPUT VOLTAGE = 12V D/C" on every SKU pictured, while the same sheet's
  spec bullets and full SKU table state 24V DC for every part number (all "-24" suffixed) —
  8th confirmed instance of the recurring 12V/24V artifact, 24V DC is correct for *this*
  document. (2) Bigger and unresolved: this directly conflicts with the existing `evenglow`
  and `evenglownicheless` cards, which — sourced from Evenglow's own install manual, before
  this sell sheet was on file — document the PCR-4/PCR-2D combo as genuinely **12V DC** for
  older Evenglow installs. Given PAL's 12V→24V driver evolution, "PCR-2D" may span an older
  12V-era board and a newer 24V-era board under the same name. Not silently resolved either
  direction — flagged in all three cards (`pcr2d`, `evenglow`, `evenglownicheless`), techs
  should confirm actual voltage on site. See "Pending sign-off" below.
- **PCR-300 Driver (LED Optics Series 3)** (`pcr300`) — built. Done. Also a 2-page sell
  sheet only (`PALPCR-300.pdf`), same coverage-gap flag as PCR-2D — no steps/DIP content
  exists to include. Two findings: (1) a real, non-artifact dual-voltage split — this
  product genuinely ships as both a 12V DC/110W ("-CL" suffix) and a 24V DC/158W ("-24"
  suffix) unit side by side, confirmed by the SKU table on both counts; explicitly flagged
  in-card as *not* the recurring 12V/24V pattern. (2) A wattage/SKU naming mismatch in the
  same spirit as the `pcr3d500` card — the sell sheet's title/header and its own cover-page
  product photo both say "PCR-300D" / "300 WATTS," but no SKU is actually rated 300W (only
  110W or 158W) — "300" reads as a platform/family number, not a real wattage, flagged
  in-card.
- **PCR-4 (Color Touch Series 4), Commander Series 2 (PC-2D), LED Optics Series 2
  (PCR-200), Power Supply Series 2 (PC-2T)** (`pcr4`, `commander2`, `ledoptics2`, `pc2t`) —
  built as minimal stub cards, photo only, no manual for any of the four. Each card's facts
  are read directly off the enclosure label visible in its own hero photo (voltage, wattage,
  certs) — nothing beyond the label is asserted, and each card carries an explicit
  "Coverage gap" line rather than inventing steps/DIP content. The PCR-4 label's own
  12V DC/50W reading independently corroborates the existing Evenglow cards' PCR-4 spec —
  no conflict there. Flagged one open question on `pc2t`: its label reads "PC-2T" (no
  "R"), 12V DC, 16W, for 2-wire lights — visibly different from the "PCR-2T-65" transformer
  the Aqualumin Replacement card requires (64- SKU family, 24V DC, higher wattage). Nothing
  on file confirms whether these are the same product under different naming or two
  genuinely different transformers — treat as unconfirmed-different, don't substitute one
  for the other on an Aqualumin job.
- All 6 new driver cards wired into the `cat-drivers` hub, the `productSelect` dropdown,
  and `HIDDEN_BY_DEFAULT` — same pattern as the rest of this batch. No live push has gone
  out for any of this batch yet, still holding per Cory's request.

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

**New (2026-07-07):** the `pcr2d` card (built from PAL's current Color Touch Series 2 sell
sheet) states 24V DC consistently for the PCR-2D driver — but the pre-existing `evenglow`
and `evenglownicheless` cards, built earlier from Evenglow's own install manual, document
the same driver as genuinely 12V DC when paired with PCR-4 for older Evenglow installs.
This is not the recurring within-document copy-paste artifact — it's two different PAL
source documents disagreeing about the same driver's real voltage. Plausible explanation
is PAL's known 12V→24V driver evolution (an older 12V-era PCR-2D vs a newer 24V-era one
under the same model name), but nothing on file confirms that. Flagged in all three
cards, not resolved either direction. Jason should confirm whether these are actually two
hardware generations, and if so around when the changeover happened, so the guide can
tell techs which one they're likely looking at from install date rather than "check
voltage and hope."

**New (2026-07-07):** the `evenglowfiberglass` card's install manual specifies a 2⅜"/2½"
holesaw for the wall-mount hole, but PAL's current sell sheet for the same product
specifies 1⅞" instead — a real diameter mismatch, not a copy-paste artifact (both figures
are stated plainly and consistently within their own document, they just disagree with
each other). Since drilling the wrong size hole is irreversible, this has **not** been
resolved either direction — the card only carries an escalate-level flag telling techs to
confirm against the actual fibreglass nut hardware in hand rather than trusting either
document blindly. Jason should confirm the correct hole diameter for this product before
a tech relies on this card for a live first-time install.

## Handoff / IP considerations (background — not an active task)
Cory needs a clean contract-exit path: PAL should be able to keep updating this guide
after the engagement ends, with no ongoing dependency on Cory's accounts/infra. Current
plan: transfer this GitHub repo to PAL's own org, and PAL links/iframes it from a
HubSpot page. This repo is intentionally simple (no backend, no API keys) so that
handoff is just a repo transfer — keep it that way unless told otherwise.
