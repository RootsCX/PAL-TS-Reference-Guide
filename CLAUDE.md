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
If `node` isn't on PATH in the working environment, macOS's built-in JavaScriptCore works
as a drop-in fallback for this same check: extract the `const DATA = [...]` block to a
file, change `const` to `var` (so `eval` leaks it into scope), then
`osascript -l JavaScript -e '...eval(fileContents); JSON.stringify({count: DATA.length})'`.

## Local preview
`.claude/launch.json` runs `ruby -run -e httpd` against a scratchpad copy of
`index.html` + `assets/`, not the project directory directly — the sandboxed preview
process doesn't have filesystem permission to read from `~/Documents/...` directly
(`Errno::EPERM`). Before previewing, rsync `index.html` and `assets/` into the session's
scratchpad dir and point `launch.json`'s `runtimeArgs` path at that copy; re-sync after
each edit you want reflected in the preview.

## Key people
- **Cory** — owner of this project, Roots CX.
- **Jason Delgado** — Florida-based field expert (installer/troubleshooter). Vets all
  new diagnostic/structural content before it goes live, and is the escalation point
  for defect-vs-installation-issue ambiguity. Anything structurally new — especially
  diagnostic decision logic — goes to Jason before being built into the HTML.

## Working principles (don't deviate from these)
- **Work autonomously; the only hard gate is push/commit (corrected 2026-07-09).**
  Don't pause to check in on routine work — crops, image analysis, validation
  commands, edits, preview verification — just do it and move on. The one rule that
  doesn't bend: never run `git commit` or `git push` without Cory reviewing the
  changes first, unless he explicitly says to just commit/push. End a turn that has
  reached that point with the checkpoint phrase "Time to review before pushing and
  committing" rather than asking permission earlier in the process.
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
- **Every nesting category must have a way back to the front page (corrected
  2026-07-09).** Any card that acts as a category/hub (has a "Select a Product" or
  "Select a Part Number" list of `product-link`s drilling into other cards) must be
  reachable from a dead end — if a user lands on it directly (search result,
  `productSelect` dropdown, or a jump from elsewhere) rather than by clicking down
  from Products, there must still be a working "back" link, not a card with no way
  out. This bit in practice when `cat-strip` (Strip Lighting) was promoted to a
  top-level front-page card (see Strip Lighting section below): the old `render()`
  logic only special-cased `directId === 'products'` for the "&larr; Back to Home"
  link, so any *other* non-hidden top-level card landed via direct jump with no back
  link at all. Fixed in `index.html`'s `render()` function (search for `Back to
  Home`) by generalizing the condition from `directId === 'products'` to
  `directId && !HIDDEN_BY_DEFAULT.has(directId)` — now *any* directly-jumped-to
  top-level (non-hidden) card gets "Back to Home", and any hidden/nested card still
  gets its normal "Back to {parent}" via `backTo`. When adding a new top-level
  category hub in the future, this now works automatically — no per-card fix needed,
  just don't add the new hub's id to `HIDDEN_BY_DEFAULT`.

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
- **PCR-5S / PCR-5CU built (2026-07-10)** — see "Automation equipment batch" below. No
  longer an open gap.

Note: the LED Bubbler (Niche Bubbler) troubleshooting card now exists in `DATA`
(id: `ledbubbler`) — no longer an open gap, and as of the Water Features refresh batch
(see below) it's a full install/troubleshooting card, not a stub. Aqualumin Replacement
(`aqualumin`), Sonar Retro Bulb (`sonarretro`), Evenglow Fiberglass
(`evenglowfiberglass`), Treo Micro (`treomicro`), and Canadian Retro (`canadianretro`)
are all now built — see batch status below. Treo Micro videos still sit unmapped in the
"Other PAL Videos" catch-all card (id: `morevideos`) rather than being attached to their
own card. Of the Sonar-branded videos in that catch-all, "PAL Sonar Remote Programming"
(×2) and "PAL Retro Lamp/Bulb Troubleshooting" / "PAL Retro Light" moved to the
`sonarretro` card, "PAL Canadian Sonar Light" moved to the `canadianretro` card, and
"PAL Concrete Bubbler" / "PAL Fiberglass Bubbler" / "PAL Bubbler" / "PAL - Pentair
Cascade Replacement Bubbler" moved to the `ledbubbler` card (all confirmed matches).
"PAL Sonar Light" (non-Canadian) remains unmapped — still unconfirmed which product it
shows.

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
- **Treo Micro (Nicheless)** (`treomicro`) — new card, built (2026-07-07). Two source
  files: an 8-page install/troubleshooting manual (`TreoMicro.pdf`) and a 2-page sell sheet
  (`PALTreoMicro-2.pdf`). Genuinely different mechanism from every other nicheless card so
  far: push-to-fit (not twist), removed with a flat-blade screwdriver rather than unscrewing
  or twisting, and it supports three distinct wall-fitting paths from one manual — concrete/
  gunite (conduit embedded during construction, no retrofit option), fiberglass (new 50mm/2"
  hole + 64-EGMIC-NA adapter, escalate-worthy first install), and vinyl (64-EGMIC-CG adapter
  threads into an *existing* standard 1.5" wall fitting — no new drilling if that fitting's
  already there). Findings: (1) the recurring 12V/24V artifact, this time appearing twice in
  one manual — the troubleshooting table says "12volts DC" and the Gunite/Concrete install
  diagram itself has both a "24VDC Transformer" label and a "12 VAC Voltage Transformer"
  callout side by side — front matter and sell sheet both confirm 24V DC is correct. (2) The
  cloning DIP table is an older 2-switch scheme (no Astral), unlike the 3-switch V3 table on
  the `drivers` card, despite both covering PCR-1Z/2Z — flagged, not merged. (3) This manual
  states PCR-2Z can power up to 60 Treo Micro lights (PCR-1Z up to 14) with a J box — much
  higher than the "up to 5 / up to 2" figures on the `treoretro` card. Not treated as a
  conflict — Treo Micro is lower-wattage, so more fit per driver — but cross-referenced both
  directions so a tech doesn't assume one number applies to the other product. Images: hero
  (`image_105.jpg`) via the same isolated-alpha-XObject technique as Sonar Retro Bulb and
  Evenglow Fiberglass; diagrams (`image_106.jpg`–`image_110.jpg`) autocropped from vector
  line art via PIL `ImageChops.difference`.
- **Canadian Retro (Sonar)** (`canadianretro`) — new card, built (2026-07-07). Last one in
  the Lights batch. Two source files: an 8-page install manual (`SonarCanadianRetro.pdf`,
  titled "Sonar Canadian Retro Light") and a 2-page sell sheet (`PALCanadaRetro.pdf`,
  titled "Sonar LED Retro-Fit Pool Light for Aqua/Lamp® Replacement" — confirms this is a
  Canadian-market retrofit for a specific competitor niche brand, "Aqua/Lamp"). Yet another
  distinct install mechanism from every other retrofit card so far: not a bulb swap (Sonar
  Retro Bulb) or a bracket+light swap onto an existing niche (Aqualumin) — this is a full
  light replacement wired on-site via a field-attachable plug (strip cable, wire into a
  screw-terminal plug block, assemble a plug housing/gland), then mounted through a new
  adapter plate that reuses the existing niche's mounting tabs. Findings: (1) voltage is
  genuinely flexible here, not the recurring artifact — the install manual consistently
  states 12V AC *or* 12/24V DC throughout, no internal contradiction — though the sell
  sheet's own feature bullet narrows it to "12V AC" only, a minor cross-document
  inconsistency flagged but not resolved. (2) No troubleshooting section exists in either
  source (same coverage gap as Aqualumin), so the card points to LED Light Diagnostics and
  the Probability-Based Framework instead. (3) No remote pairing/cloning button sequence is
  documented for this product specifically, even though the sell sheet confirms Hayward/
  Jandy/Pentair cloning support via the same general "Sonar" remote family used on
  Aqualumin and Sonar Retro Bulb — flagged as unconfirmed rather than assumed identical.
  Moved "PAL Canadian Sonar Light" out of the `morevideos` catch-all onto this card
  (confirmed match on manual title) and updated the cross-reference notes on `aqualumin`
  and `sonarretro`. Images: hero (`image_111.jpg`) via the same isolated-alpha-XObject
  technique as the other Sonar-platform cards; diagrams (`image_112.jpg`–`image_116.jpg`)
  autocropped from vector line art via PIL `ImageChops.difference`.
- **Lights batch complete.** Evenglow through Canadian Retro are all built and pushed live
  (commit cc8515f).

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
  and `HIDDEN_BY_DEFAULT` — same pattern as the rest of this batch. Pushed live.

## Automation integrations
New top-level category, separate from the Lights and Drivers/Controllers batches above —
covers third-party pool automation platforms that integrate with PAL lighting directly
(as opposed to the Pentair/Hayward/Jandy protocol-cloning path documented on the
`competitors` card). Added a `cat-automation` hub (parallel to `cat-lights`/`cat-drivers`/
etc.) under the `products` hub.

- **The Attendant (Poolside Tech)** (`attendant`) — new card, built 2026-07-07. Source:
  `source-manuals/Automation/Poolside Tech/` — a knowledge-base article ("Connecting PAL
  Lighting to The Attendant") saved as both HTML and a 15-page print-to-PDF, from
  poolside.support (Poolside Tech LLC, a separate company from PAL that makes a pool
  automation platform called "The Attendant"). Unlike every other card in this guide,
  this one documents controlling PAL lights over native **DMX512** addressing rather than
  brand-protocol cloning — only PAL's DMX-capable drivers (PCR-2DMX, PCR-3DMX) work with
  it; every other PAL driver still needs the Competitors card's cloning path instead.
  Images: the print-to-PDF's embedded images were print-paginated/downsampled (some split
  across a page break), so instead of extracting from the PDF, pulled the same images at
  full original resolution directly from their poolside.support CDN URLs embedded in the
  saved HTML file (`data-orig-file` attributes) — up to 3015×2319, much cleaner than the
  PDF versions. 13 images total (`image_117`–`image_129`), resized to a 1600px max
  dimension and saved as JPEG.
  - **Cross-reference finding, not yet acted on:** the article's "PCR-3DMX (Old Style)"
    section (8 zones, DIP switches 9-1) documents what appears to be the same physical
    product as this guide's `pcr3dmx8z` card. More notably, the "64-PCR-3DMX V3" board
    photo in the article — showing Module Config / Board Function switch banks with
    Cloner / Module / DMX / Strip modes — matches the newer V3 "High Powered" board that's
    flagged as not-yet-built-out on the `pcr3d500` and `pcr3dmx8z` cards, pending Jason's
    review (see Pending Sign-off below). This article documents that V3 board's **DMX
    mode** specifically, from a working third-party integration already in the field —
    real supporting evidence for that pending review. Deliberately **not** merged into the
    `pcr3d500`/`pcr3dmx8z` cards' content — flagged only, per the standing "structurally
    new diagnostic content goes to Jason first" rule.
  - **Flag — DIP polarity inconsistency within the source article itself:** of the four
    driver variants documented (PCR-2DMX Old Style, PCR-3DMX Old Style, PCR-2DMX V3,
    64-PCR-3DMX V3), three read UP=ON/DOWN=OFF and one — PCR-3DMX (Old Style), the 8-zone
    variant — reads the opposite (DOWN=ON/UP=OFF). Not resolved either direction; flagged
    in-card so a tech doesn't carry one driver's polarity convention over to another.
  - The render logic's `issues` field always prints a hardcoded "Light Not Turning On —
    check in order" section label (see `cardHTML()` in index.html) — accurate for the
    light-product cards it was designed for, but wrong for this card's DMX-signal
    troubleshooting content. Worked around by hand-authoring the troubleshooting table
    inside `extra` with its own correctly-worded `section-label` instead of using the
    `issues`/`issuesNote` fields. Pre-existing render behavior, not changed — same
    workaround would apply to any future non-light-product card that needs a
    cause/action table.
  - Wired into `cat-automation`, the `productSelect` dropdown, and `HIDDEN_BY_DEFAULT` —
    same pattern as the rest of the guide. Pushed live (commit b6c8bb5).

## Strip Lighting batch (in progress)
New products added under the existing `cat-lights` hub (not a new category) — Cory's
`source-manuals/Strip/<Product>/` folder covers PAL's Evenglow linear-strip family in
three product tiers: Quick Ship Strip, Perimeter Strip Kits, and Custom Strip. Building
one at a time, pausing for Cory after each.

All three tiers share the exact same 30-page `INSTALLATION_GUIDE.pdf` (confirmed
byte-identical file across the Quick Ship Strip and Custom Strip folders) — a masonry/
mounting-scenario reference (pool coping x3 methods, corners & curves, track install,
steppingstones, infinity edge, water/fire bowls, garden beds, fire feature walls, swim
outs/baja shelves, hardscapes, water features, spa toe kick, stair installation), not a
per-product troubleshooting manual. It has no electrical-fault/troubleshooting section at
all. Product-specific facts (electrical specs, SKUs, wattage tables, driver compatibility)
come from each tier's own spec sheets/brochure instead.

- **Quick-Ship Strip Lighting (NFWB8/NFLT8)** (`quickshipstrip`) — new card, built
  2026-07-07. Two source spec sheets (`PAL-NFLT8-Spec-Sheet.V26.2.pdf`,
  `PAL-NFWB8-Spec-Sheet.V26.2.pdf`) plus a 2-page tier brochure
  (`Quick-Ship-Strip-Lighting-Brochure-USA-V2.pdf`). This tier is PAL's pre-assembled,
  factory-terminated, fixed-stocked-length product (4ft–60ft) — explicitly **not**
  field-cut, which is the main thing that distinguishes it from the still-unbuilt Custom
  Strip card. Two physical series covered by one card since they're the same product tier
  or just different bend/mount orientation: NFWB8 ("Side View," side-bending, 238°
  beam) and NFLT8 ("Top View," top-bending, 112° beam) — same electrical family, same
  driver compatibility (PCR-1Z/PCR-3D families, shared with the `drivers` and
  `pcr3d500` cards), different mounting tracks. Confirmed via the spec sheet's own
  wiring diagram that strip lighting runs off the identical PCR-1Z board/DIP-cloning
  layout (Jandy/Pentair/Hayward/PAL/Astral) already documented on the `drivers` card —
  cross-referenced, not duplicated. Three findings, all flagged in-card rather than
  resolved:
  1. **Power-consumption conflict** — both spec sheets state "3 W/FT," but the Quick-Ship
     brochure states "2.5W per Ft" for both series. The brochure's own per-length wattage
     table is arithmetically consistent with the 3W/ft figure (60ft NFLT8 = 183W ≈
     3.05W/ft), not 2.5W/ft, but neither document was silently preferred in the card text.
  2. **Driver wattage/SKU-suffix conflict** — the Quick-Ship brochure's Drivers table
     lists 64-PCR-1Z-65/64-PCR-3D-120/64-PCR-3D-500 as 55W/100W/400W, while the spec
     sheets' own Remote Driver reference lists the identical SKUs at 65W/120W/500W
     (i.e. suffix = wattage). Two PAL documents disagree about the same SKUs' rated
     wattage.
  3. **Cutting-increment conflict** — the shared 30-page install guide states Evenglow
     Flex cuts in increments of "50mm (2\")" in one section (Over-Runs) and "99mm (3.9\")"
     in another (Steppingstone Entry Points), while the spec sheets separately state
     "1\" (25mm)." Three different figures across PAL's own documents at the time this
     card was first built. **Update (Perimeter Strip Kits research, below):** two more
     NFWB8/NFLT8-specific sources (the dedicated Perimeter Strip Kit install manual and
     PAL's in-field cut/reseal instructions) both independently confirm 1" (25mm) too —
     four sources now agree. The card's flag was updated to note this as "leaning
     resolved" — the general guide's 50mm/99mm figures likely describe a different strip
     variant, not this product family, though that hasn't been directly confirmed.
  Also carried forward as a real in-workflow escalation instruction (not a build-process
  note): PAL's install guide states any submerged strip installation must have its design
  submitted to design@pallighting.com for sign-off before install — added as an
  `escalate` box.
  Images: hero (`image_130.jpg`) cropped from the NFLT8 spec sheet's own clean product
  photo. `image_131.jpg` (NFWB8-vs-NFLT8 mounting/bend comparison) and `image_132.jpg`
  (general driver-to-strip wiring diagram) came from the spec sheets at 300dpi.
  `image_133.jpg` (entry-point dark-spot wrong/correct setting) and `image_134.jpg`
  (corner bend-radius/overhang clearance) came from the shared install guide, also at
  300dpi. Ordering/SKU/driver-compatibility data was transcribed as real HTML tables
  rather than images, since it's plain numeric reference data — more searchable and
  accessible than a table baked into a JPEG.
  Render-logic note: like the `attendant` card, this card uses the `issues` field
  normally (not worked around) — most of its issues are genuinely light/strip-appearance
  symptoms (dark spot, cracked at a corner, dim segment), so the hardcoded "Light Not
  Turning On" section label is a reasonable-enough fit here, unlike the DMX-only
  `attendant` card.
  Wired into `cat-lights`, the `productSelect` dropdown, and `HIDDEN_BY_DEFAULT`. Pushed
  live (commit 8517844, along with the rest of the Strip Lighting batch).
  **Corrected 2026-07-07** after building the Perimeter Strip Kits card below, using
  evidence found in that research: (1) the "not field-cut" claim was an overstatement —
  PAL's own in-field cut/reseal instructions explicitly list Quick-Ship SKUs as an
  applicable product, so it CAN be field-cut/resealed at the installer's risk (facts text
  and the corresponding issue reworded, not deleted — Custom Strip is still pointed to for
  a job planned around a non-stocked length from the start); (2) the power-consumption
  flag was strengthened to note the Perimeter Strip Kit sell sheet independently
  corroborates 3W/ft (3 of 4 PAL documents now agree, vs. the Quick-Ship brochure's
  outlier 2.5W/ft); (3) the non-submersible language was sharpened with the specific "15
  sustained minutes" threshold quoted in the Perimeter Strip Kit manual (same product
  family, same non-submersible strip); (4) the cutting-increment flag was updated per the
  4-source resolution above. A cross-reference line to the new Perimeter Strip Kits card
  was also added at the end of `extra`.
- **Perimeter Strip Kits (NFWB8/NFLT8)** (`perimeterstripkit`) — new card, built
  2026-07-07. PAL's all-in-one, longer-length tier (85'/100'/115') of the same Evenglow
  Strip/Track family as Quick-Ship — one kit contains everything needed for a full
  perimeter run: LED strip pre-installed on a reel dispenser, PC mounting track, end caps,
  end-cap glue, a remote driver (transformer + controller combined) with remote, and a
  pre-installed 100ft 4-wire power cable. Four source files, richest of the strip batch so
  far: a 2-page sell sheet (`Pool-Perimeter-Strip-Lighting-Kits-USA-Jan29-2026.pdf`, full
  kit-options/ordering/UPC/driver tables), a dedicated 12-page Install & Operations Manual
  (`Perimeter-Strip-Kit-US-InstallationGuide-Print.pdf` — unlike Quick-Ship, this tier has
  its own real Safety/Delivery/Tools/Before-Installation/Installation/Troubleshooting
  manual, not just the shared 30-page masonry-scenario guide), a 2-page in-field
  cut/reseal instruction sheet + warranty T&Cs
  (`PAL-Lighting-Infield-Strip-Cutting-Instructions-with-warranty-VFeb-26.pdf`, explicitly
  lists both Quick-Ship and Perimeter Strip Kit SKUs as applicable), and confirmed
  (via md5) that the NFLT8/NFWB8 spec sheets in this folder are byte-identical to the
  already-processed Quick-Ship copies. Findings:
  1. **Field-cutting is expected here, not an edge case** — a Perimeter Strip Kit is sized
     to a specific pool perimeter, so the install manual's own Step 6C treats cutting to
     fit as a normal step, unlike Quick-Ship where it's an off-label (if PAL-sanctioned)
     workaround. Same risk/warranty language applies: installer assumes all risk, and an
     incorrectly-sealed joint is excluded from the 3-year warranty.
  2. **Power consumption (3W/ft) and cutting increment (1"/25mm)** — both independently
     corroborated by this tier's own sell sheet/manual, strengthening (not just repeating)
     the same figures already flagged on the Quick-Ship card — see that card's updated
     flags above.
  3. **Driver is a dual-zone board (Z1/Z2)** with universal 100-240V AC input (two loose
     leads, Brown=active/hot, Blue=neutral) and a specific power-cable wire-color mapping
     confirmed off the manual's own wiring diagram: Red-Red / Blue-Blue / White-White /
     Grey-Green (Grey represents the green LED channel — not obvious from the wire color
     alone, worth calling out explicitly). Same PCR-1Z/PCR-3D driver family as Quick-Ship
     and the `drivers`/`pcr3d500` cards, but a full-length kit run (255-345W) specifically
     needs the 400W-class driver (64-PCR-3D-500/-3DW-500) — the 55W/100W-class units in
     that same family are undersized for any stocked kit length on their own. This kit's
     own manual doesn't reproduce the Cloning DIP switch table for the board pictured, so
     the card points to the `drivers` card for that table rather than guessing.
  4. **Troubleshooting content is thin by design** — no bulleted fault table exists in the
     dedicated manual beyond a "call or email us" instruction, though it does explicitly
     acknowledge automation-compatibility issues (e.g. "a Pentair automation system is
     faulty and causing our LED strip to operate incorrectly") as a real, expected support
     scenario — cross-referenced to the `competitors` and `attendant` cards. Flagged
     in-card (`issuesNote`) that the issues list is inferred from the manual's own
     warnings and this guide's shared diagnostic framework, not copied from a dedicated
     table — same pattern already used on the `pcr2d`/`pcr300` coverage-gap cards.
  Also carried forward the same submerged-install design-approval `escalate` box as
  Quick-Ship (identical source warning, same product family).
  Images: hero (`image_135.jpg`, kit-contents illustration: reel/tracks/driver) and all
  install diagrams (`image_136.jpg` pool perimeter/channel/conduit planning,
  `image_137.jpg` corner bend-radius/curve-spacing, `image_138.jpg` 2-person install
  technique with the "do not push with screwdriver" warning, `image_139.jpg` safe-to-cut
  point diagram, `image_140.jpg` resealing-sequence diagram, `image_141.jpg` remote driver
  wiring diagram) all came from the dedicated 12-page install manual at 300dpi, cropped
  via row/column whitespace-band inspection against the full rendered page rather than
  guessed pixel boxes — this workflow hit two page-index mistakes along the way
  (`image_136` and `image_138` were each initially cropped from the wrong page and had to
  be re-rendered/re-cropped after Read-tool verification caught the wrong content) —
  every crop in this card was verified with the Read tool before being treated as final.
  Wired into `cat-lights`, the `productSelect` dropdown, and `HIDDEN_BY_DEFAULT`,
  positioned directly after `quickshipstrip`. Pushed live (commit 8517844).
- **Custom Strip Lighting (Design-Build)** (`customstrip`) — new card, built 2026-07-07.
  Last one in the Strip Lighting batch. Confirmed via md5 that the Custom Strip folder's
  `INSTALLATION_GUIDE.pdf` is byte-identical to Quick-Ship's copy (same shared 30-page
  masonry/mounting-scenario guide, already used) — only new source was a 2-page sell sheet
  (`Custom-Strip-Sales-Sheet-USA-Jan-29-2026.pdf`). This tier turned out to be materially
  bigger in scope than expected — not just "Quick-Ship's NFWB8/NFLT8 cut to a custom
  length." It's a project-scoped design-build service (free PAL design consultation,
  layout, strip built to the job's exact spec, photography credit for selected projects)
  covering **nine physical variants across three mounting categories**: Side View (NFWB8,
  NFLS8, NFMS2, NFUS2), Top View (NFLT8, NFMT8, NFUT2), and Hardscape (PLO2-F, PLO2-W) —
  transcribed as a full comparison table rather than an image, same "real tables over
  baked-in images" preference used on the sibling cards' ordering tables. Findings:
  1. **NFLS8 vs NFLT8 naming collision** — one letter apart (S=Side, T=Top) in an
     otherwise near-identical SKU pattern, both "Flex" strip products in the same family.
     Flagged prominently as the single most likely misread on a work order.
  2. **First submersible strip products in the whole guide** — NFUS2 and NFUT2 are
     genuinely IP68 submersible-rated, unlike every other strip variant across all three
     cards (Quick-Ship, Perimeter Strip Kits, and this card's other 7 variants), which all
     carry the non-submersible/15-sustained-minute warning. The shared install guide's own
     "must read" boilerplate independently corroborates this by explicitly carving out an
     exception for "underwater rated LED strip lights." Flagged so a tech doesn't reflexively
     repeat the non-submersible warning to a customer who has one of these two SKUs.
  3. **Rigid-track Hardscape line (PLO2-F, PLO2-W)** — a fundamentally different mounting
     mechanism from the other 7 flexible variants: a rigid track with mitred corners for
     direction changes, not a bendable curve-spaced track. No install manual is on file
     for this specific mounting system — flagged as a coverage gap rather than assuming
     the flexible-strip bend-radius/curve-spacing guidance from the other cards applies.
  4. **No per-length wattage table** — unlike Quick-Ship and Perimeter Strip Kits, this
     sell sheet doesn't publish one (the product is project-quoted). Flagged not to assume
     the 3W/ft figure resolved on those two cards applies uniformly across all 9 variants
     here, especially the physically different Mini/Underwater/Hardscape lines.
  5. NFWB8 and NFLT8 specifically are the same physical products already documented on the
     Quick-Ship and Perimeter Strip Kits cards — cross-referenced (three purchase paths
     for the same two variants), not treated as a conflict.
  No dedicated troubleshooting section exists in the source sell sheet (same coverage-gap
  pattern as `pcr2d`/`pcr300`/Perimeter Strip Kits) — issues list is inferred from the
  shared diagnostic framework and the sibling cards, flagged via `issuesNote`.
  Images: hero (`image_142.jpg`) cropped from the sell sheet's own hardscape-scene product
  photo (had to crop tightly around overlapping bullet-point text baked into the same
  image layer — two earlier crop attempts still caught stray text before landing on a
  clean product-only region). `image_143.jpg`/`144.jpg`/`145.jpg` (Side View / Top View /
  Hardscape product-line diagram groups) are direct crops of the sell sheet's own
  three category sections at 300dpi, each verified against the full page render before
  being treated as final.
  Wired into `cat-lights`, the `productSelect` dropdown, and `HIDDEN_BY_DEFAULT`,
  positioned directly after `perimeterstripkit`.
- **Strip Lighting batch complete.** Quick-Ship, Perimeter Strip Kits, and Custom Strip
  are all built and pushed live (commit 8517844).

## Custom Strip SKU-first restructuring (complete, 2026-07-09)
Cory asked to convert `customstrip` from one combined 9-variant card into a real
sub-hub — same "hub links to individual product cards" pattern already used for
`cat-drivers` and `cat-water` (as opposed to a single card covering every SKU). Pilot
case before doing the same to Remotes/Drivers/Lights per Kazi's whiteboard "Guide
List" (see prior open thread). **All 9 SKU cards are now built and wired in** — see
the "Remaining 6 SKU cards built in one continuous pass" entry below for the final
batch and the open items still pending Cory's input.

- **`customstrip`** — converted to a thin hub (matches `cat-drivers`/`cat-water`:
  no `photo`, no `steps`/`issues`/`escalate`). First pass kept the NFLS8/NFLT8
  naming-collision and NFUS2/NFUT2 submersible-exception flags at hub level, plus a
  "Custom Design Process" list — **Cory corrected this as too cluttered/confusing**:
  removed both flag note-boxes and the Design Process list entirely, and reordered
  so the Product Line Comparison table sits directly under the intro paragraph, with
  "Select a Part Number" directly below the table (was previously flags → part-number
  list → table → design process). Final hub shape: intro facts → comparison table →
  reference links → part-number list. **Feedback captured:** this user finds stacked
  amber note-boxes/flags on a routing-only hub page cluttering rather than helpful —
  keep hub pages to comparison data + navigation, put flags on the product card where
  the tech is actually working the call, not on the router page.
  Added a **Wattage column** to the comparison table (see wattage correction below)
  and a **Reference** line linking PAL's live-hosted PDFs: the Strip Lighting
  Installation Guide (`https://pallighting.com/manuals/strips/INSTALLATION_GUIDE.pdf`)
  and the NFWB8 Spec Sheet (`https://pallighting.com/wp-content/uploads/2026/02/
  PAL-NFWB8-Spec-Sheet.V26.2.pdf`) — both provided directly by Cory, placed on the hub
  per his explicit instruction (not duplicated onto the NFWB8 card itself).
  **Still a judgment call, not yet confirmed with Cory:** the 8 SKUs other than NFWB8
  have no dedicated card yet and are only reachable via the comparison table — only
  add a link to "Select a Part Number" once a SKU actually has a card.
- **`customstrip-nfwb8`** — first SKU card built, pilot for the new per-SKU template.
  Source: `Custom-Strip-Sales-Sheet-USA-Jan-29-2026.pdf` +  `INSTALLATION_GUIDE.pdf`
  (shared 30-page masonry guide, byte-identical to Quick-Ship's/Perimeter's copies,
  no NFWB8-specific content). Confirmed NFWB8 here is the **same physical strip** as
  the Quick-Ship Strip and Perimeter Strip Kits cards' NFWB8, bought through the
  design-build/project-quoted path instead — install technique, DIP cloning table,
  and driver options are cross-referenced to those two cards rather than duplicated.
  Reused 4 existing diagram assets instead of re-extracting: `image_133.jpg`/
  `image_134.jpg` (entry-point dark spot, corner bend radius — from `quickshipstrip`)
  and `image_139.jpg`/`image_140.jpg` (safe-cut-point, resealing sequence — from
  `perimeterstripkit`).
  **Corrected after Cory's review (photo + content placement):** hero `photo` went
  through two rounds — first swapped from the small isolated glow-icon
  (`image_175.jpg`) to the full spec-box crop (`image_176.jpg`, matching Cory's
  reference screenshot), then swapped again to a new side-by-side composite
  (`image_177.jpg`, see below) once Cory pointed at richer per-SKU source assets he'd
  added. `image_176.jpg` is still used, now as a zoomable step-1 reference image
  rather than the hero. Removed the `issuesNote` field ("No dedicated troubleshooting
  section exists...") — flagged by Cory as build-process/meta commentary that
  shouldn't appear in the guide (same standing "no build-process talk inside
  index.html" rule, apply it to `issuesNote` fields too, not just `note`/`extra`).
  Moved the "NFWB8 Quick Specs" table from the bottom (`extra`) up into `facts`,
  directly under "Driver compatibility" — pushes "Light Not Turning On"
  troubleshooting further down the card, which was the intent.
  **Wattage/cutting-increment correction (2026-07-09, confirmed by Cory):** initial
  build stated 3W/ft, 1" (matching the figure already resolved across 4 PAL
  documents on the Quick-Ship/Perimeter cards). Cory then relayed 6 wattage figures
  from Jason including two labels absent from any Custom Strip source file —
  "NFWB2" (4W/ft, 2") and "NFLT2" (4W/ft, 2"). Asked Cory directly whether these were
  shorthand for NFWB8/NFLT8 or separate products; **he confirmed via explicit
  multiple-choice: NFWB2/NFLT2 = Jason's shorthand for NFWB8/NFLT8, and the 4W/ft
  (2") figures should overwrite the previous 3W/ft (1") ones.** Updated this card's
  facts, Quick Specs table, driver-sizing math, and the field-cut step to 4W/ft / 2"
  (50mm) throughout.
  **Flag box removed per Cory's direction — see [[feedback_no_incard_flags]]:** the
  initial correction added an in-card amber note-box explaining the conflict between
  this new 4W/ft figure and PAL's own printed NFWB8 spec sheet (3W/ft, 1") and the
  still-unchanged Quick-Ship/Perimeter cards. Cory said to stop putting flags like
  that in front of the team entirely — "that all stays in the background for you and
  I" — so the box was removed and the facts/step text now states 4W/ft, 2" plainly
  with no caveat visible to a tech. **The underlying conflict is NOT resolved, only
  hidden from the card** — full detail stays here: PAL's own NFWB8 spec sheet/install
  guide and the `quickshipstrip`/`perimeterstripkit` cards (documenting this same
  physical SKU) still say 3W/ft, 1"; this card now says 4W/ft, 2" on Jason's
  authority via Cory. Whether to update `quickshipstrip`/`perimeterstripkit` to match
  — and whether that reopens the 4-source "resolved" conclusion already documented on
  `quickshipstrip` — still needs a decision from Cory; flagged to him, not actioned.
  **New source-manuals structure found, use for remaining 8 SKU cards:** Cory added
  `source-manuals/Strip/Custom Strip/<Family>/<SKU>/`, one subfolder per mounting
  family exactly matching the hub's 3 categories — `Downward Facing_Side Bending`
  (NFWB8, NFLS8, NFMS2, NFUS2 = Side View), `Outward Facing_Top Bending` (NFLT8,
  NFMT8, NFUT2 = Top View), `Hardscape_Outward Facing` (PLO2-F, PLO2-W = Hardscape).
  Each SKU folder has a clean transparent product render (`*-Prod.png`), a dimension
  diagram with both inch and mm values (`*-Dim.png`/`*-DIMS.png`), and — new, richer
  than what was used to build the original 9-variant combined card — an
  **individual per-SKU spec sheet PDF** (e.g. `PAL-NFWB8-Spec-Sheet.V26.2.pdf`,
  `PAL-NFMS2-Spec-Sheet.V26.2.pdf`; PLO2-F and PLO2-W share one `PAL-PLO2-Spec-
  Sheet.V26.2.pdf`). This is what Cory meant by "the PAL-TS-Reference Guide as I've
  structured it" — use this per-family/per-SKU folder layout (not the old combined
  sell sheet) as the source for each of the remaining 8 SKU cards.
  **New hero-photo pattern for this batch:** composited each SKU's `*-Prod.png` +
  `*-Dim.png` side by side (white background, thin gray divider, no captions — same
  technique as the `drivers` card's two-enclosure composite) rather than using a
  sell-sheet crop or a single image. NFWB8's is `image_177.jpg`. Use this same
  composite pattern for the remaining 8 SKU cards, sourced from each one's own
  `Prod.png`/`Dim.png` pair.
  **Wired** into the hub's part-number list, the `productSelect` dropdown, and
  `HIDDEN_BY_DEFAULT`.
  **Corrected again (2026-07-09):** removed the NFWB8 Spec Sheet link from the hub's
  Reference line — Cory wants only the general Strip Lighting Installation Guide
  linked there for now; he'll provide the correct per-SKU spec sheet link for each
  part number once that SKU's card actually exists, rather than linking NFWB8's spec
  sheet on the hub where it doesn't clearly belong to one product. The NFWB8 spec
  sheet link belongs on the `customstrip-nfwb8` card itself instead — added directly
  under the "Driver compatibility" bullet, right before "NFWB8 Quick Specs" (his
  explicit placement). **Pattern for the remaining 8 SKU cards:** each one's own spec
  sheet link (from its `source-manuals/Strip/Custom Strip/<Family>/<SKU>/PAL-<SKU>-
  Spec-Sheet*.pdf` folder — Cory will provide the live pallighting.com URL per SKU)
  goes in that same spot on its own card — Driver compatibility bullet → spec sheet
  link → Quick Specs table. Also made hero
  `photo` images zoomable guide-wide: added the `zoomable` class + `cursor:zoom-in`
  to the shared `photo` rendering in `cardHTML()` (index.html, the `if(item.photo)`
  line) — this was a global render-logic change, not a data change, so it applies to
  every card's hero photo automatically, not just `customstrip-nfwb8`. Cory asked for
  this specifically because `image_177.jpg` carries real dimension text that's hard
  to read at the 260px card width; tapping now opens the same lightbox already used
  for every other diagram in the guide.
- **`customstrip-nflt8`** — second SKU card built (2026-07-09), pilot confirmation
  that the individual per-SKU spec sheets are materially richer than what built the
  original combined 9-variant card and richer than what was available for NFWB8 at
  build time. Source: `Outward Facing_Top Bending/NFLT8/PAL-NFLT8-Spec-Sheet.V26.2.pdf`
  — a single well-organized page (vs. the old cramped 9-per-page combined sheet) with
  sections the old source never had: Lifespan (L70 &gt;84,000 hrs @ 25&deg;C),
  Operating temp (-40&deg;F to 122&deg;F), Max length (115'/35m), an **IK08 impact
  rating** alongside IP68 (not previously documented on any strip card), a full
  RGB + DMX remote driver compatibility table (64-PCR-1ZW-65, -2ZW-65, -3DW-120,
  -3DW-500 / 64-PCR-2DMX-65, -3DMX-120, -3DMX-500, -3DMX-500-8Z), and — genuinely
  new troubleshooting-relevant content — a **5-option Cable Entry diagram** (rear,
  straight, side-left, side-right, front) governing how the driver cable physically
  connects to the strip/track. Added a new issue row for this ("driver cable won't
  seat cleanly — confirm which of the 5 entry configurations was ordered") since nothing
  like it existed on `customstrip-nfwb8`.
  **Wattage:** 4W/ft, 2" (50mm) — same Jason-confirmed figure already on the hub's
  comparison table for NFLT8, applied the same way as NFWB8 (no in-card flag; this
  SKU's own printed spec sheet states 3W/ft, 1" instead, same recurring conflict,
  tracked here only per [[feedback_no_incard_flags]]).
  **Bending radius reported differently than NFWB8:** this individual sheet gives one
  figure (&gt;2.36"/60mm) rather than NFWB8's two-figure RGB/SPI-DMX split — used
  as-is rather than forcing NFWB8's format onto it, since this source doesn't make
  that distinction.
  **Not yet done:** spec sheet link intentionally omitted from this card — Cory
  hasn't provided the live pallighting.com URL for NFLT8 yet (same placement slot as
  NFWB8's, once available: Driver compatibility bullet → spec sheet link → Quick
  Specs table).
  Images: `image_178.jpg` (hero — NFLT-Prod.png + NFLT-DIMS.png composite, same
  technique as NFWB8's `image_177.jpg`) and `image_179.jpg` (new — Cable Entry
  Options diagram, cropped from the individual spec sheet's own page render at
  300dpi). Reused `image_133.jpg`/`image_134.jpg` (entry-point/corner diagrams) and
  `image_139.jpg`/`image_140.jpg` (cut-point/resealing diagrams) from the sibling
  Quick-Ship/Perimeter cards, same as NFWB8.
  Wired into the hub's part-number list, `productSelect`, and `HIDDEN_BY_DEFAULT`.
  **Open follow-up, flagged to Cory, not yet actioned:** NFWB8's own individual spec
  sheet (`Downward Facing_Side Bending/NFWB8/PAL-NFWB8-Spec-Sheet.V26.2.pdf`) almost
  certainly has this same richer Lifespan/Operating-temp/IK-rating/driver-table/cable-
  entry content — `customstrip-nfwb8` was built before this was discovered and only
  used the older combined 9-variant sheet for its facts text (the individual sheet
  was used solely for the hero composite photo). Worth a pass to backfill NFWB8 to
  the same depth once the remaining SKUs are done, if Cory wants parity.
- **`customstrip-nflt8` spec sheet link added (2026-07-09):** Cory provided
  `https://pallighting.com/wp-content/uploads/2026/02/PAL-NFLT8-Spec-Sheet.V26.2.pdf`
  — added directly under the "Driver compatibility" bullet, right before "NFLT8
  Quick Specs" (same slot as NFWB8's).
  **Build order specified by Cory (SKU-card sequence, not build-session order):**
  NFWB8 → NFLS8 → NFMS2 → NFUS2 → NFLT8 → NFMT8 → NFUT2 → PLO2-F → PLO2-W — this is
  the order the hub's "Select a Part Number" list should read in once everything is
  built (matches the Product Line Comparison table's row order, which was already in
  this sequence). NFWB8 and NFLT8 were already built out of this order (NFWB8 first,
  NFLT8 second) — their relative order on the hub list is still correct as-is (1st
  and 5th), nothing to fix yet; just means the remaining 7 need to slot into their
  correct relative positions as each is built, not simply appended after NFLT8.
  **All 7 remaining spec sheet links, provided by Cory in this same message —
  use each one on its own card in the same slot (Driver compatibility bullet → spec
  sheet link → Quick Specs table) as that card gets built:**
  - NFLS8: `https://pallighting.com/wp-content/uploads/2026/02/PAL-NFLS8-Spec-Sheet-V26.2.pdf`
  - NFMS2: `https://pallighting.com/wp-content/uploads/2026/02/PAL-NFMS2-Spec-Sheet.V26.2.pdf`
  - NFUS2: `https://pallighting.com/wp-content/uploads/2026/05/PAL-NFUS-V2-Spec-Sheet-5-26.pdf`
  - NFMT8: `https://pallighting.com/wp-content/uploads/2026/02/PAL-NFMT8-Spec-Sheet.V26.2.pdf`
  - NFUT2: `https://pallighting.com/wp-content/uploads/2026/05/PAL-NFUT-V2-Spec-Sheet-5-26.pdf`
  - PLO2-F: `https://pallighting.com/wp-content/uploads/2026/02/PAL-PLO2-Spec-Sheet.V26.2.pdf`
  - PLO2-W: `https://pallighting.com/wp-content/uploads/2026/02/PAL-PLO2-Spec-Sheet.V26.2.pdf`
    (PLO2-F and PLO2-W share the identical URL — one combined spec sheet PDF covers
    both SKUs, consistent with them sharing one source file in `source-manuals/Strip/
    Custom Strip/Hardscape_Outward Facing/` too.)
- **`customstrip-nfls8`** — third SKU card built (2026-07-09), inserted into the
  `DATA` array between `customstrip-nfwb8` and `customstrip-nflt8` (not appended at
  the end) to match Cory's specified final order — same treatment applied to the
  hub's part-number list and the `productSelect` dropdown. Source:
  `Downward Facing_Side Bending/NFLS8/PAL-NFLS8-Spec-Sheet-V26.2.pdf`. Key facts:
  113&deg; beam / 37 lm/ft — genuinely narrow compared to NFWB8's 238&deg; "Wide
  Beam," which is the real distinguishing feature behind the NFWB8/NFLS8 naming
  collision already flagged on the hub; called this out explicitly in facts and
  step 1 so a tech isn't just told "different SKU" without knowing *how* different.
  **No Jason wattage override applies here** — Jason's 6 relayed figures only covered
  NFWB8/NFLT8/NFUS2/NFUT2 (after the NFWB2&rarr;NFWB8/NFLT2&rarr;NFLT8 shorthand
  resolution), not NFLS8, so this card uses its own printed spec as-is: 3W/ft, 1"
  (25mm) — no conflict, nothing to override.
  **Cross-reference corrected, not copy-pasted from NFWB8/NFLT8:** those two cards
  can truthfully say "same physical strip as Quick-Ship/Perimeter" because NFWB8 and
  NFLT8 are also sold through those other two tiers. **NFLS8 has no Quick-Ship or
  Perimeter Strip Kit equivalent — Custom Strip is its only purchase path** — said so
  explicitly rather than reusing the sibling cards' cross-reference language
  unchanged, which would have overclaimed a stocked/kit option that doesn't exist
  for this SKU. The mounting *technique* cross-reference to Quick-Ship/Perimeter
  still stands (same strip family mechanics), just not a SKU-identity claim.
  **Cable entry options differ from NFLT8 — only 4, not 5:** this spec sheet's own
  Cable Entry Options section shows Rear/Straight/Side-Left/Side-Right only, no
  Front entry (NFLT8 has all 5). Cropped and used as its own diagram rather than
  assuming NFLT8's set carried over. **No end cap SKU found** for this specific SKU
  in any source on file (unlike NFWB8/NFLT8, which have one via the Quick-Ship/
  Perimeter cards' own accessories tables) — left out of the Quick Specs table
  rather than guessing one by pattern-matching the `64-PAL-NFLS-MT-E-6` track SKU
  naming, and flagged in an order-note so a tech doesn't assume it's just missing
  from this card.
  Images: `image_181.jpg` (hero — NFLS-Prod.png + NFLS-DIMS.png composite) and
  `image_180.jpg` (Cable Entry Options diagram, cropped from this SKU's own spec
  sheet at 300dpi, confirmed only 4 options before cropping). Reused
  `image_133.jpg`/`image_134.jpg`/`image_139.jpg`/`image_140.jpg` from the sibling
  cards, same as NFWB8/NFLT8.
  Wired into the hub's part-number list (correct position, between NFWB8 and
  NFLT8), `productSelect`, and `HIDDEN_BY_DEFAULT`.
- **Remaining 6 SKU cards built in one continuous pass (2026-07-09), no per-card
  pause.** Cory said to keep going through the rest of the batch autonomously
  ("build the rest of the SKU cards in the exact same way? Keep going") rather than
  stopping after each — consistent with [[feedback_autonomy_gate_at_push]]. All 9
  Custom Strip SKU cards now exist: NFWB8, NFLS8, NFMS2, NFUS2, NFLT8, NFMT8, NFUT2,
  PLO2-F, PLO2-W — Custom Strip is fully built out. Each inserted at its correct
  position in the `DATA` array, the hub's part-number list, and `productSelect`
  (verified in preview: hub lists all 9 in the exact specified order). Validated
  after each insertion; full browser check at the end (console clean, no failed
  network requests, all 16 new image assets verified non-corrupt).
  - **`customstrip-nfms2`** (Mini Side View) — `Downward Facing_Side Bending/NFMS2/
    PAL-NFMS2-Spec-Sheet.V26.2.pdf`. Notably smaller cross-section (5/16"&times;1/2")
    than NFWB8/NFLS8. No Jason wattage override (not in his 6-item list) — own
    printed spec used as-is: 3.5W/ft, 2" (50mm). Two real oddities in this specific
    document, presented as printed without inventing a flag (per
    [[feedback_no_incard_flags]]) but worth knowing about: (1) Lifespan L70 &gt;
    60,000 hrs @ 25&deg;C — lower than the Series 8 strips' 84,000 hrs, a real
    Mini/Series-2-vs-Series-8 family difference, not an error; (2) Operating temp
    printed as -40&deg;C to <b>110&deg;C</b> (-40&deg;F to 230&deg;F) — unusually
    high vs. every other card's ~50&deg;C ceiling, but the C&harr;F conversion is
    internally consistent (not a garbled-digit typo), so used as printed rather than
    second-guessed. No end cap SKU published — omitted rather than guessed.
  - **`customstrip-nfus2`** (Underwater Side View) — `Downward Facing_Side Bending/
    NFUS2/PAL-NFUS-V2-Spec-Sheet-5-26.pdf`. One of only two genuinely submersible
    Custom Strip SKUs. **No wattage conflict at all** — this document's own printed
    spec (2W/ft, 2"/50mm) matches Jason's figure exactly, unlike the NFWB8/NFLT8
    case. New real fact worth flagging: mounting track ships in 3' sections
    (`64-PAL-NFUS2-MT-E-3`), not the standard 6' used everywhere else — noted
    plainly in facts/steps. IK10 impact rating carries a real installation
    precondition (custom track must be PAL-design-team-approved before install) —
    kept in the card since it's actionable, not a discrepancy narration. Escalate
    box rewritten from scratch (not just SKU-swapped from the non-submersible
    template) since this SKU's logic is the exception, not the rule.
  - **`customstrip-nfmt8`** (Mini Top View) — `Outward Facing_Top Bending/NFMT8/
    PAL-NFMT8-Spec-Sheet.V26.2.pdf`. No Jason override; own printed spec used:
    3W/ft, 1" (25mm). Mounting track SKU is `64-PAL-NFMT2-MT-E-6` — uses "NFMT2" in
    the track code, not "NFMT8" — third confirmed instance of a track SKU using a
    different number than its strip (after NFWB2/NFWB8 and the earlier
    hypothesis about Jason's NFWB2/NFLT2 shorthand). Additional supporting evidence
    for that still-open theory, not yet confirmed with Jason. 5 cable entry options
    (matches NFLT8's Top View pattern, incl. Front entry).
  - **`customstrip-nfut2`** (Underwater Top View) — `Outward Facing_Top Bending/
    NFUT2/PAL-NFUT-V2-Spec-Sheet-5-26.pdf`. **Confirmed copy-paste artifact**: this
    document's own Ordering Guide / Accessories sections (SERIES/LUMINAIRE TYPE box,
    mounting track SKU `64-PAL-NFUS2-MT-E-3`) are copy-pasted from the NFUS2 sheet
    wholesale — say "NFUS2 / LED Strip - Submersible - Side bending" even though
    this document's own title/header/hero photo are unambiguously NFUT2 (top-bending,
    outward-facing). Confirmed by direct visual inspection of the rendered page, not
    just text extraction. **Mounting track SKU omitted from this card** rather than
    reproducing the wrong SKU or guessing the correct one. **New unresolved wattage
    question, flagged to Cory, not yet confirmed:** this SKU's own printed spec says
    1.7W/ft, 1" (25mm) — doesn't match Jason's 2W/ft, 2" figure for NFUT2 (unlike
    NFUS2, which matched exactly). Used Jason's figure anyway, applying the same
    precedent Cory already confirmed for NFWB8/NFLT8 (field figure overrides printed
    spec when they conflict) — but Cory hasn't explicitly re-confirmed this
    extension for NFUT2 specifically, since his original confirmation was about the
    NFWB2/NFLT2 naming question, not this. Worth a direct check with him.
  - **`customstrip-plo2f`** and **`customstrip-plo2w`** (rigid Hardscape "Feature
    Strip" line) — both share one spec sheet,
    `Hardscape_Outward Facing/PLO2-F(or W)/PAL-PLO2-Spec-Sheet.V26.2.pdf` (confirmed
    byte-identical earlier this session). Genuinely different product mechanism from
    every flexible SKU on this hub: rigid track, mitred corners, no bend-radius
    concept. Real distinguishing spec: **IP65, not IP68** — a lower ingress rating
    than every flexible variant, reflecting the rigid encapsulated design (not a
    copy-paste artifact, confirmed consistent within this document). Cable entry is
    **Straight entry only** — no rear/side/front alternative. No cut-point/reseal
    diagram or install manual exists for this rigid mounting system in any source on
    file — flagged as a coverage gap in both cards' steps rather than assuming the
    flexible-strip cut/reseal procedure applies. F vs. W is purely a track
    cross-section difference (4/5"&times;3/4" vs. 7/16"&times;3/4") — identical
    strip/electronics otherwise; steps/issues call out the naming collision risk
    explicitly since "PLO2-F" and "PLO2-W" are visually easy to transpose.
  - Images: `image_182`–`image_187` (hero composites, same Prod+Dim technique as
    NFWB8/NFLT8/NFLS8) and `image_188`–`image_191` (Cable Entry Options diagrams for
    NFMS2/NFMT8/NFUS2/NFUT2, cropped from each SKU's own spec sheet at 300dpi — NFUS2
    and NFUT2 required different crop coordinates since their document layout places
    Cable Entry Options in the left column rather than the lower-right like the
    Series 8 sheets). PLO2-F/PLO2-W have no cable-entry diagram (Straight-entry-only,
    nothing to illustrate) and reuse no corner/entry-point/cut-point diagrams either,
    since none of that flexible-strip content applies to a rigid track.
- **Custom Strip hub is now fully built out — all 9 SKU cards exist.** Only open
  items: (1) confirm the NFUT2 wattage question with Cory (see above); (2) decide
  whether to propagate the NFWB8/NFLT8 4W/ft correction to the still-unchanged
  `quickshipstrip`/`perimeterstripkit` cards (open since the NFWB8 card was built,
  still not actioned); (3) whether to backfill `customstrip-nfwb8` with the same
  Lifespan/Operating-temp/driver-table depth its own individual spec sheet has,
  now that every other SKU card uses that richer sourcing (open since the NFLT8
  card was built, still not actioned).
- **Strip Lighting promoted to the true front page, not nested under Products
  (2026-07-09, two-step correction).** Was nested under `cat-lights` (Quick-Ship,
  Perimeter Strip Kits, and Custom Strip all showed as three of the ten links on the
  Lights hub). First pass moved it to a new `cat-strip` hub listed as a category
  inside the `products` hub (same level as Lights/Drivers/Water Features in that
  hub's own list) — **Cory corrected this**: "It should not be nested under
  Products" — he wanted it out of the category-picker entirely and shown directly on
  the actual home/landing view, alongside `framework`/`competitors`/`led-diagnostics`/
  `products` themselves, not one click deeper. Final state: removed the "Strip
  Lighting" link from `products`'s own list; removed `backTo: "products"` from
  `cat-strip` (top-level cards like `products`/`framework`/`competitors` don't carry
  a `backTo`); removed `cat-strip` from `HIDDEN_BY_DEFAULT` so it renders on the
  default (no-search) view like any other top-level card — its position in the
  `DATA` array (right after `products`, before the now-hidden `cat-drivers`) means it
  naturally lands as the 5th card on the home page, directly under Products, with no
  array reordering needed. `quickshipstrip`/`perimeterstripkit`/`customstrip` still
  point `backTo: "cat-strip"`, and that back-link resolves correctly regardless of
  `cat-strip`'s own hidden/visible status — `HIDDEN_BY_DEFAULT` only gates whether a
  card shows on first load, not whether it can be a valid `backTo` target. Tapping
  "Strip Lighting" on the home page now expands it in place (same accordion behavior
  as `products`) to reveal the three tiers; Strip → Custom Strip → SKU nesting below
  it is unchanged. Verified in preview: home page shows Strip Lighting as its own
  card (not nested), expands correctly, and the full Custom Strip → NFWB8 back-link
  chain still resolves.

## Water Features refresh batch (in progress)
New source manuals under `source-manuals/Water Features/<Product>/` — Bubblers,
WaterSphere, Waterblade. Unlike the Lights/Drivers/Strip batches, these three products
already existed as stub-ish cards in `DATA` (`ledbubbler`, `watersphere`, `waterblade`)
from earlier work with no image extraction behind them — this batch is a refresh (real
manual content + images), not new-card creation, except where a product turns out to be
genuinely undocumented. Same PyMuPDF + Pillow @350dpi workflow, numbered
`assets/image_NN.jpg`, flag discrepancies in-card rather than resolving them. Proceeding
one product at a time, pausing for Cory after each.

- **LED Bubbler (Niche Bubbler)** (`ledbubbler`) — refreshed 2026-07-07. Three source
  files: a 24-page Concrete/Vinyl install & operations manual
  (`PAL-Bubbler-ConcreteVinyl-InstructionsR00-2.pdf`), a 2-page sell sheet
  (`PALBubbler-2.pdf`), and a 1-page plume-height/water-pressure reference
  (`BubblerWaterPressureGuide.pdf`). The card previously had no photo and no install
  content at all (facts + troubleshooting only) — added a hero photo, full Concrete and
  Vinyl/FG install-diagram sections, the plume-height table, and a Vinyl/FG replacement
  parts table. Troubleshooting steps were already accurate against this same manual from
  earlier work and were left as-is. Findings, all flagged in-card rather than resolved:
  1. **Two copy-paste artifacts bleeding in from the WaterSphere manual**, not corrected
     in PAL's own source: the Bubbler manual's IMPORTANT NOTICE section repeats
     WaterSphere's sun-shelf warning verbatim ("The PAL WaterSphere is only for
     gunite/concrete installations..."), and the Concrete "Installing Luminaire" diagram
     has a callout reading "Optional lens (only available for 600mm and 800mm Spheres)" —
     a WaterSphere globe-size reference with no relevance to the Bubbler's own ¾"/1.2"
     spout lenses. Both read as reused boilerplate from the WaterSphere manual, not real
     Bubbler content.
  2. **Section-heading mislabel** — the manual's own Section 5 (driver/DIP-switch
     hardware install) is titled "INSTALLATION FOR GUNITE/CONCRETE POOLS," identical to
     Section 3's title, apparently copy-pasted from that earlier section header. Same
     reused-template pattern already seen on other cards (e.g. PCR-3DMX-8Z, Evenglow
     Fiberglass) — flagged, not corrected.
  3. **Within-manual pairing-procedure inconsistency** — Section 6's troubleshooting
     "resync the remote" steps say press Code Setting once then tap Z1 three times, while
     Section 5's own "Features" list documents a different official "Matching Code"
     procedure (apply power, press Z1 once within 5 seconds) for what appears to be the
     same action. This is the same open question already flagged on the `drivers` card
     (see Pending sign-off) — cross-referenced rather than re-litigated here.
  4. **Real (non-artifact) install-depth difference** — Concrete requires the lens no less
     than 1" below normal water level; Vinyl/FG requires 18" — a large, easy-to-conflate
     difference between the two versions of the same product, called out explicitly in
     facts and in each install section.
  5. **Voltage** — this manual's own IMPORTANT NOTICE and driver spec consistently state
     24V DC with no internal contradiction, but the current PAL sell sheet's "Design &
     Features" bullet reads "Low Power / 12/24V DC" — another instance of the recurring
     12V/24V ambiguity pattern seen on Drivers/PCR-2D/PCR-3D-120/500, flagged in-card.
  6. **Coverage gap** — the manual's Replacement Parts section (Section 7) only tables
     parts for the Vinyl/FG version; no equivalent table exists for Gunite/Concrete.
     Flagged rather than inventing part numbers.
  Cross-checked DIP-switch/cloning content against the `drivers` card — identical
  PCR-1Z/2Z table, so not duplicated here; card points to Drivers instead.
  Images: hero (`image_146.jpg`) is an isolated alpha-masked cutout of the sell sheet's
  own exploded light/lens product photo (same XObject+smask technique used on the Sonar
  Retro Bulb/Evenglow Fiberglass/Treo Micro/Canadian Retro cards), composited onto white
  and autocropped. `image_147`–`151` are install-diagram pages from the manual, cropped
  to content bounds via whitespace-band detection and stacked into per-section composites
  (concrete: niche/conduit/coverage, cable/render-ring, luminaire/wiring; vinyl/FG:
  drill-hole/niche, cable/luminaire) so callout-connected diagrams aren't split across
  separate images. `image_152` is the Vinyl/FG replacement-parts exploded diagram only
  (cropped separately from its own table, which was transcribed as a real HTML table
  instead per the project's plain-numeric-data preference). Plume height/water-pressure
  data also transcribed as a real table rather than an image.
  No changes needed to `productSelect`, hub links, or `HIDDEN_BY_DEFAULT` — this card
  already existed and was already wired in from earlier work.
- **WaterSphere** (`watersphere`) — refreshed 2026-07-07. Two source files: a 15-page
  install & operations manual (`PALWatersphereInstructions-24-32inch.pdf`) and a 4-page
  sell sheet (`WATERSPHERE-Brochure-24-32inch-2.pdf`). Confirmed the WaterSphere has no
  electronics of its own — it's an acrylic globe that mounts on an adapter on top of an
  LED Bubbler, and all lighting/electrical content is the Bubbler's. The install manual's
  Sections A-D (niche/electrical install) are near-identical to the Bubbler manual's own
  Sections A-D — same physical niche/collar hardware — so those steps were **not**
  duplicated here; the card cross-references the `ledbubbler` card instead and only
  documents what's genuinely Sphere-specific: Sphere Preparation (E), Fixing the Sphere
  to the Bubbler (F), the plumbing Overview, and Replacement Parts. This also resolved a
  question from the Bubbler refresh: the Bubbler manual's "Optional lens (only available
  for 600mm and 800mm Spheres)" callout, initially flagged there as a possible copy-paste
  artifact, is confirmed **accurate** — this manual's own Replacement Parts diagram shows
  the same optional Lens Adapter — so that flag was corrected on the `ledbubbler` card.
  The sun-shelf warning bleeding into the Bubbler manual's IMPORTANT NOTICE, however, is
  confirmed as a genuine copy-paste artifact — this manual is its actual source.
  Findings, all flagged in-card rather than resolved:
  1. **Three-way SKU/kit-contents conflict** — the install manual's own cover page lists
     `64-EGBSP-24`/`-32` ("with Concrete Mounting Bracket," no cord-length variants); the
     sell sheet instead lists `64-EGBSP-CGS-080/150-24/-32`, described as including the
     Bubbler light and cable; but the install manual's own Package Contents and
     Replacement Parts sections both list the PAL Evenglow Bubbler as sold separately
     (part `64-EGB-CGS-XXX`), box contents being just Sphere + Adapter + 6 screws. Not
     resolved — card tells techs to confirm against the actual packing slip rather than
     assume either document.
  2. **Nominal vs. actual size** — the "32-inch" Sphere is actually 800mm/31.5" diameter
     per the sell sheet's own dimension diagram, a real (if minor) rounding gap between
     the marketing name and the physical part. Included in facts so a tech isn't thrown
     by a customer's tape-measure reading.
  3. **No troubleshooting section exists** in this manual at all (ToC is Safety /
     Preparation / Installation / Replacement Parts only) — same coverage-gap pattern as
     `pcr2d`/`pcr300`/Perimeter Strip Kit; issues list flagged via `issuesNote` as
     inferred, not sourced from an official table.
  Also surfaced a real diagnostic improvement over the old stub card: the manual's own
  Sphere Preparation section gives PolyWatch (scratches) and Anti-Fog Spray (haze) as the
  first-line fix for a scratched/hazy globe — the old card jumped straight to "replace
  the globe." Issues list now sequences PolyWatch/Anti-Fog before replacement.
  Images: hero (`image_153.jpg`) is an isolated alpha-masked cutout of the sell sheet's
  own clean product photo (same XObject+smask technique as the Bubbler and other recent
  cards), tightened to a stricter alpha threshold after an initial crop kept a wide band
  of soft drop-shadow on one side. `image_154.jpg` is the stack-order diagram with the
  sell sheet's own QR code cropped out (not useful in this guide and not something we
  control the destination of). `image_155.jpg` stacks the Sphere Preparation and
  Fixing-to-Bubbler pages. `image_156.jpg` is the full plumbing overview page (kept as
  one image since the diagram, both multi-sphere plumbing layouts, and the GPM/warning
  text are all on one connected page). `image_157.jpg` is the Replacement Parts exploded
  diagram; adapter/part data was legible enough to fold into facts text rather than
  needing a separate transcribed table.
  No changes needed to `productSelect`, hub links, or `HIDDEN_BY_DEFAULT`.
- **Water Features refresh batch: Bubbler and WaterSphere complete.** Waterblade
  (`waterblade`) — already a fuller card from earlier work with its own images
  (`image_18`-`image_21`) — has not yet been checked against the newer
  `source-manuals/Water Features/Waterblade/` folder; still open for this batch.

## WiFi / Remotes / Color Touch App restructure (2026-07-08)
The `cat-wifi` hub (title now "WiFi, Remotes, Color Touch App", per Cory's explicit
labeling request) now links three distinct cards instead of one combined card:

- **Remotes** (`remotes`) — built earlier this batch, unchanged here. Side-by-side
  reference for the PCZ-2, PCT-1, and two visually-different "Sonar" remotes, each
  reusing diagrams already extracted for their product cards (`image_70`, `image_02`,
  `image_62`, `image_95`) rather than new extractions. Every remote still lives on its
  own product card too — this is additive, nothing was removed from those cards.
- **Wi-Fi** (`wifi`) — narrowed to network/module hardware only (was previously titled
  "Wi-Fi & Color Touch App" and mixed network setup with app content). Added a hero
  photo of the physical module (`image_169.jpg`, Part No. 64-WIFI, isolated
  alpha-masked cutout) documenting its three status LEDs (Power On / WiFi Not
  Connected / WiFi Connected) and Reset button — this wasn't photographed anywhere in
  the guide before. Confirmed the 2.4GHz-only requirement directly against the app's
  own Router Connect screen text ("cannot support 5G router").
- **Color Touch App** (`colortouchapp`) — new card, built from
  `source-manuals/WiFi, Color Touch App, Remotes/ColorTouchApp.pdf` (18-page official
  screenshot walkthrough). Covers account creation/login, connecting a new driver to
  Wi-Fi, updating app/driver firmware, setting up a new driver, day-to-day operation
  (zones, RGB, brightness, shows, music sync), renaming zones, and scheduling. Moved
  the three existing YouTube videos here from the `wifi` card (they're about app
  usage, not module hardware). Images `image_158`–`image_168`, one per workflow
  section, direct crops of the source PDF's own slides (already full-bleed, no
  cropping needed) — some single-page, some two-page composites where the source
  splits one workflow across two slides.
  - **Finding — pairing procedure discrepancy, not resolved:** the `wifi` card
    previously documented linking a driver via "tap '+' → select '2.4g' → driver
    appears, select it → in-app go to 'Touch 2' → tap 'Link' → on the transformer,
    press Code Setting, then press Link (bottom-left) 3x." This new official guide's
    own screens instead show holding the module's physical **Reset** button for 3
    seconds until the light flashes, then finishing entirely in-app (WiFi password +
    Start Configuration) — no Code Setting/Link button step appears anywhere in it.
    Flagged on the `wifi` card rather than silently overwritten — could be an older
    app/module version, or two different scenarios (first-time setup vs. re-linking
    an already-configured driver). Leads with the Reset-button procedure as current.
  - **Finding — Touch 5 / Touch 9 drivers surfaced, not documented anywhere in this
    guide:** the app's own "Setting Up a New Driver" screen offers four driver types —
    1ZW, 2ZW, Touch 5, Touch 9 (shown in-app as "Color Touch 1/2/5/9"). 1ZW/2ZW match
    the existing PCR-1ZW/2ZW drivers on the `drivers` card. Touch 5 and Touch 9 are
    new to this guide — `source-manuals/Automation/Pool Touch 5/` and
    `.../Pool Touch 9/` folders exist but haven't been reviewed. Per the standing
    "structurally new content goes to Jason first" rule, **not** built out here —
    only flagged on the `colortouchapp` card so a tech isn't caught off guard if a
    customer's app shows one of these two. See Pending Sign-off below.
  - Confirmed the app's own zone/driver naming ("TOUCH-2") matches the existing
    `wifi` card's older reference to going to "Touch 2" in-app — that part of the old
    text wasn't wrong, just incomplete against the fuller picture this new source
    gives.
- Wired `colortouchapp` into `cat-wifi`, the `productSelect` dropdown, and
  `HIDDEN_BY_DEFAULT`. Updated the top-level `products` hub's category link and tags
  to match the new "WiFi, Remotes, Color Touch App" label.

## Remotes card expansion (2026-07-09)
Cory found `source-manuals/WiFi, Color Touch App, Remotes/Remotes/` — a folder with a
subfolder per remote SKU (product page PDF + isolated product photo, occasionally a
manual), one folder per SKU Kazi's team whiteboard-audited. This confirmed PAL lists
**6 remote handset SKUs** total, matching the whiteboard almost exactly: 42-PCT-1T,
42-PCT-3T, 42-PCT-5T (all flagged "Disc." on the whiteboard), 64-PCZ-2, 64-PAL-SR (also
"Disc." on the whiteboard), and 64-PAL-SR2 ("newest"/4-channel per the whiteboard) — the
one SKU with **no folder on file**, confirming the whiteboard's "not in guide" note isn't
just a documentation gap, there's no source material for it at all yet.

Rebuilt the `remotes` card from 4 sections to 7, one per confirmed SKU:
- **PCZ-2 (64-PCZ-2)** and **PCT-1 (42-PCT-1T)** — already documented; added each one's
  new clean isolated product photo (`image_174.jpg`, `image_170.jpg`) alongside the
  existing labeled diagrams. Confirmed 42-PCT-1T's official title covers "Color Touch
  Series 1/2/3/4 Controllers" — one remote spans four controller generations, not
  previously called out.
- **PCT-3 (42-PCT-3T)** — new section, new product. "Remote Handset for Commander Series
  2 Controllers," branded "Commander Touch" on the unit (`image_171.jpg`). Pairs with the
  2-wire Commander Series 2 (PC-2D) driver — this is the first confirmed remote
  cross-reference for that card, which previously stated "not yet seen cross-referenced
  by name in any other card in this guide." Updated `commander2`'s facts to name it.
  CH1/CH2 + mode buttons visible on the unit, but no pairing/cloning steps document was
  provided — flagged as a coverage gap rather than guessing a button sequence.
- **PCT-5 (42-PCT-5T)** — new section, new product. "Remote Handset for Pool Touch – 5
  Automation System," branded "Touch-5" (`image_172.jpg`). This is real, physical
  evidence that Pool Touch 5 is a genuine PAL product — cross-referenced from the
  `colortouchapp` card's existing Touch 5/Touch 9 flag as supporting evidence. Per the
  standing "structurally new goes to Jason first" rule, the Touch 5 automation/driver
  logic itself is still **not** built out — only the remote handset is documented here.
  Same coverage-gap flag as PCT-3 (no pairing steps on file).
- **Sonar wand-style (64-PAL-SR)** — already documented; added its own clean product
  photo (`image_173.jpg`) alongside the existing diagram, confirmed official title.
- **Sonar rounded-style (Aqualumin)** — unchanged content, but added an explicit
  open-question note: this 4-channel, no-Astral remote may actually **be** the missing
  64-PAL-SR2 (whiteboard describes SR2 as "newest," 4-channel, replacing the 8-channel
  SR) — plausible given the channel-count match, but **not confirmed** against a real
  part number or physical label, so it wasn't renamed. Flagged for confirmation rather
  than assumed.
- Added a **"discontinued vs. spare-parts-only" note** to the card's facts: the
  whiteboard marks 42-PCT-1T/3T/5T and 64-PAL-SR as "Disc.," but PAL's own site (product
  pages captured 2026-07-09) still lists all of them live under "Remote Handsets, Spare
  Parts" — worded so a tech doesn't read "discontinued" as "can't be ordered," since
  these are still legitimate replacement parts for an existing install, just not what
  you'd spec for a new job.
Images `image_170`–`image_174`: each is an isolated-alpha PNG already provided (not
extracted from a PDF render this time — PAL's own product folders included clean
transparent-background photos directly), composited onto white and autocropped the same
way as every other hero photo in this guide.
No changes to `productSelect`, hub links, or `HIDDEN_BY_DEFAULT` — `remotes` already
existed and was already wired in.

**Resolved 2026-07-09:** Cory confirmed via a screenshot of the exact same "Section 4.
REMOTE PROGRAMMING" diagram (already on file as `image_62.jpg`) that the rounded
Aqualumin remote **is** the 64-PAL-SR2 — same image, so no new extraction needed, just
relabeling. Updated the `remotes` card: section renamed "Sonar Remote — SR2 style
(64-PAL-SR2)," part number field filled in (was "not confirmed"), and added that it
operates the same way as the 64-PAL-SR (same pairing/cloning/unpairing procedure shape,
just 4 channels instead of 8, no Astral) per Cory's explicit ask. Cross-referenced from
the SR (discontinued) section too, so the "replaced by" relationship reads correctly in
both directions.

**Corrected 2026-07-09** — two follow-up fixes from Cory after reviewing the first pass:
1. **Removed the in-card "open question" note-box** about the SR2/rounded-remote identity
   match. Per the standing "no build-process talk inside index.html" rule, internal
   tracking language ("the team tracks a 6th SKU," "no source manual... has been provided
   yet") doesn't belong in front of a tech on a live call — it's tracked here in CLAUDE.md
   only (see above), not duplicated in the guide itself.
2. **Standardized every remote section to one template**: photo first, then a short
   Part Number / What It Is / Pairs With line, then any unique facts/issues — replacing
   the previous free-form paragraph-first layout. Also **reordered the whole card**:
   active remotes (PCZ-2, the rounded Sonar remote, Color Touch App) now come first,
   followed by all four discontinued remotes (PCT-1, PCT-3, PCT-5, Sonar wand-style SR),
   each with its section-label styled in the existing `--escalate` red (`#e3556b`) and an
   explicit "— Discontinued" suffix so it reads correctly even without color (e.g. on
   print or for colorblind accessibility). Reused the site's existing escalate-box red
   rather than inventing a new color.

## Drivers & Controllers SKU-first restructuring (active SKUs complete, 2026-07-09)
Cory's direction: **Custom Strip's hub+SKU-card pattern is now the gold standard** for
every product category in the guide, not just Strip Lighting. Drivers & Controllers is
the pilot case for applying it elsewhere. Real motivation, not just consistency-for-its-
own-sake: the team fields a lot of calls on **discontinued** drivers with very little
documentation behind them — Cory wants an eventual "Discontinued" branch under Drivers
once the active-SKU restructuring is done. Kazi supplied two whiteboard photos
(2026-07-09) with a full driver/remote SKU audit — same effort as the Remotes card
expansion audit above, this time covering Active Drivers, 2-Wire Drivers, and
Discontinued Drivers.

Built one pilot SKU card (`pcr1z65`) first and paused for review, per
[[feedback_autonomy_gate_at_push]]. Cory reviewed it and said to continue through the
rest of the active-SKU batch in one pass — same "keep going" pattern as the Custom Strip
SKU batch — with one explicit instruction: **break up the bundled "driver family" cards
completely; every SKU gets its own card**, not just the pilot.

**`cat-drivers` promoted to the front page** — same mechanical treatment as the
`cat-strip` promotion: removed the "Drivers & Controllers" link from the `products` hub's
own category list, removed `backTo: "products"` from `cat-drivers`, removed `cat-drivers`
from `HIDDEN_BY_DEFAULT`. Lands naturally as the 6th top-level card (right after Strip
Lighting), no array reordering needed.

**Every active SKU from the whiteboard's 13-item list now has (or already had) its own
card.** The two bundled family cards — `drivers` (5 SKUs: 1Z-65/1ZW-65/1Z-SM-65/2Z-65/
2ZW-65) and the old `pcr3d500` (4 SKUs: 3D-120/3DW-120/3D-500/3DW-500) — were split:
- **`drivers`** kept its id but was **retitled and trimmed into a shared reference page**:
  "PCR-1Z / PCR-2Z Family — Cloning, Pairing & Zone Reference." Removed the SKU-bundling
  "Part numbers:" facts line (that content now lives on each SKU's own card); kept
  everything genuinely shared — both Cloning DIP tables (V2 4-switch, V3 3-switch), the
  Matching/Clearing Code pairing procedure with board/remote diagrams, the Zone/Color-
  Temperature "Other Config" bank, the Automation Clone Mode relay note, and the videos —
  completely unchanged, so no real content was lost. No longer listed under "Select a
  Part Number" on the `cat-drivers` hub (it isn't a SKU); moved to a new "Shared
  Reference" section instead.
- **4 new leaf cards built:** `pcr1zw65` (Wi-Fi), `pcr1zsm65` (switch mode — called out
  its V3 3-switch cloning table as a real differentiator from its remote-capable
  siblings' 4-switch table, since that's an easy mix-up), `pcr2z65` (dual-zone — kept the
  "double-tap Z1/Z2 to sync zone colors" field-confirmed tip inline since it's the single
  most common call on this SKU, not just cross-referenced), `pcr2zw65` (dual-zone Wi-Fi).
  Each cross-references the reference card for the DIP/pairing detail rather than
  duplicating it, same convention as `customstrip-nfwb8` → `quickshipstrip`.
- **Old `pcr3d500` id freed and reused**: the bundled 4-SKU card was renamed to id
  `pcr3dref` / title "PCR-3D Family — Mounting, Cloning & Configuration Reference" (same
  trim-not-delete treatment — mounting/wiring diagrams, Cloning DIP table, Zone/Color-Temp
  table, Automation relay note, V3-board flag, and fan-filter maintenance all kept
  unchanged). The freed `pcr3d500` id was then given to the actual new leaf card for
  64-PCR-3D-500, so cleaner-looking ids didn't require inventing awkward suffixes.
- **4 new PCR-3D leaf cards built:** `pcr3d120`, `pcr3dw120`, `pcr3d500` (now the actual
  500W remote SKU, not the family), `pcr3dw500`. Each is intentionally short — mounting/
  wiring/DIP content is byte-identical across the whole PCR-3D family in the source
  manual, so every leaf card cross-references `pcr3dref` rather than re-embedding the
  same diagrams four times. Carried the existing wattage/SKU-naming-mismatch flag forward
  onto the 500W and 3DW-500 cards specifically, since it's most relevant there (the
  original source manual never confirmed a "-500" suffix, only "-300" — the whiteboard is
  the only source confirming 64-PCR-3D-500/64-PCR-3DW-500 as currently active).
- **`cat-drivers` hub's "Select a Part Number" list rebuilt fully flat** — all 9 new SKU
  cards plus the two still-single-SKU-equivalent cards (`pcr2dmx`, `pcr3dmx8z`) plus the
  unrelated single-SKU-ish cards (`pcr2d`, `pcr300`, `pcr4`, `commander2`, `ledoptics2`,
  `pc2t`, untouched, out of this batch's scope — see below), plus a new **Shared
  Reference** section linking the two reference cards. Comparison table's "Documented on"
  column updated to point at the correct individual card for every one of the 13 active
  SKUs.
- **Hero photos:** all 9 new leaf cards reuse the existing family photos
  (`assets/image_65.jpg` for the PCR-1Z/2Z family, `assets/image_77.jpg` for PCR-3D) since
  this batch's only new source material was a whiteboard photo, not new manual PDFs — no
  dedicated per-SKU photography exists yet, unlike the Custom Strip SKU cards which had
  dedicated Prod/Dim PNGs per SKU from Cory. Worth a dedicated photo pass later if Cory
  wants per-SKU imagery here too.
- **Verified in preview:** DATA parses (66 entries, no duplicate ids, no missing ids),
  hub expands showing the rebuilt comparison table and full flat SKU list, every one of
  the 11 driver-related cards (5 PCR-1Z/2Z + 1 reference + 4 PCR-3D + 1 reference) resolves
  with a working "← Back to Drivers & Controllers" link, all 52 `productSelect` dropdown
  entries resolve to real content, all referenced images return 200 OK, no console errors.

**Not touched, out of scope for this pass:** `pcr2dmx` and `pcr3dmx8z` already map 1:1 to
a single active SKU family each (the whiteboard's wattage-tier SKUs under `pcr2dmx`, and
the single 8-zone SKU under `pcr3dmx8z`) — no bundling to break up. `pcr2d`, `pcr300`,
`pcr4`, `commander2`, `ledoptics2`, `pc2t` don't appear on the whiteboard's Active Drivers
list at all (they're older/Color-Touch-Series platforms outside this audit's scope) —
left exactly as they were.

**Known follow-up, not urgent:** roughly 15 other cards across the guide (Strip Lighting
tiers, Custom Strip SKUs, WaterSphere, PCR-300, the Attendant, cloning-competitor card,
etc.) contain **prose** references to "the Drivers card" or "the PCR-3D-120/500 card" by
name — these are plain text, not `jumpToProduct()` links, so nothing is broken, but
they're now slightly imprecise (the content they're pointing at is now split across a
reference card and several SKU cards). Not worth a mass-edit pass on its own; fine to
touch up opportunistically whenever one of those cards is next edited for another reason.

**Whiteboard data captured verbatim (source of truth for what still needs a card):**
- **Active Drivers (13, all cloning-capable):** 64-PCR-1Z-65, 64-PCR-1ZW-65,
  64-PCR-1Z-SM-65, 64-PCR-2Z-65, 64-PCR-2ZW-65, 64-PCR-3D-120, 64-PCR-3DW-120,
  64-PCR-3D-500, 64-PCR-3DW-500, 64-PCR-2DMX-65, 64-PCR-3DMX-120, 64-PCR-3DMX-500,
  64-PCR-3DMX-500-8Z. Whiteboard notes a search-tuning observation, not a content fact:
  "1Z"/"2Z" only surfaces these in search under certain terms — not acted on, just
  recorded in case search weighting is revisited later.
- **2-Wire Drivers (3, separate from the 13):** 64-PCR-2T-65 (no remote, white light
  only — in guide only via the Aqualumin retrofit page), 64-PCR-3T-120 (not in guide),
  64-PCR-3T-500 (not in guide) — two genuine, confirmed content gaps.
- **Discontinued Drivers (5), per Kazi's second whiteboard:** 4-wire — 42-PCR-2D (in
  guide as `pcr2d`), 42-PCR-4 (whiteboard marks "NO"), 42-PCR-200 (whiteboard marks
  "NO"). 2-wire — 42-PC-2D (whiteboard marks "NO"), 42-PCR-8A "Commander" (whiteboard
  marks "NO").
- **Generic discontinued-driver troubleshooting** (whiteboard, for "12VDC output"
  discontinued drivers): 1) check for power, 2) press/release the S1 button on the board
  (manual override), 3) if neither restores the light, find the replacement part number —
  treated as non-serviceable once discontinued. Also noted: "most common lights = Treo
  2T → replace w/ Treo Max" — see open question below, not resolved.

**Open discrepancies flagged, not resolved:**
1. Whiteboard marks 42-PCR-4 and 42-PCR-200 as "NO" (not in guide), but minimal stub
   cards already exist for both (`pcr4`, `ledoptics2` — "LED Optics Series 2 (PCR-200)").
   Likely the whiteboard audit only counted cards with real troubleshooting depth, not
   photo-only stubs — worth clarifying with Kazi/Cory whether stub-level coverage should
   count as "in guide" going forward, since it affects how the discontinued-drivers phase
   gets scoped.
2. **Three-way (possibly four-way) SKU-naming confusion around 2-wire transformers,**
   compounding an already-open question: the existing `pc2t` card's own enclosure label
   reads "PC-2T" (no "R"); the Aqualumin Replacement card requires "PCR-2T-65"; the
   existing `commander2` card's title references "PC-2D"; the whiteboard's discontinued
   list separately lists "42-PC-2D" and "42-PCR-8A 'Commander'" as two distinct rows, both
   marked "NO." Nothing on file confirms how many actual distinct products are tangled
   under PC-2T / PCR-2T-65 / PC-2D / PCR-8A naming — not resolved, needs Cory/Jason before
   any of this is built into discontinued-driver content.
3. 64-PCR-3DMX-120 / 64-PCR-3DMX-500 (no "-8Z" suffix) are confirmed active part numbers
   per the whiteboard, but the only existing card in that family (`pcr3dmx8z`) is
   explicitly 8-zone-specific throughout (channel map, board diagrams). Flagged on the new
   `cat-drivers` comparison table as "not yet documented" rather than assumed covered.
4. "Treo 2T" (from the discontinued-drivers whiteboard's troubleshooting notes) doesn't
   match any product name anywhere else in this guide or in PAL's source manuals reviewed
   so far (closest names: Treo Max+, Treo Retro, Treo Mini+, Treo Micro) — flagged, not
   guessed at.

**Active-SKU restructuring is done.** All 11 SKUs from the whiteboard that had existing
content now have their own individual card (`pcr1z65`, `pcr1zw65`, `pcr1zsm65`, `pcr2z65`,
`pcr2zw65`, `pcr3d120`, `pcr3dw120`, `pcr3d500`, `pcr3dw500`, plus the already-1:1
`pcr2dmx` and `pcr3dmx8z`), backed by two shared reference cards (`drivers`, `pcr3dref`).

**Next steps (not started):**
- Build the two confirmed-but-undocumented active SKUs (64-PCR-3DMX-120/500 non-8Z,
  64-PCR-3T-120/500) once their content is sourced — real content gaps, not a
  restructuring task.
- Discontinued-drivers phase: a "Discontinued" dropdown/branch under Drivers &
  Controllers, covering the 5 SKUs from the second whiteboard plus the generic 12VDC
  troubleshooting sequence (check power → press/release S1 manual-override button → find
  replacement part number) — deliberately deferred until Cory resolves the PC-2T/PCR-2T-65/
  PC-2D/PCR-8A naming confusion flagged above, since building discontinued content on top
  of an unresolved SKU-identity question would risk documenting the wrong product.
- Decide whether the same full SKU-split treatment should extend to `pcr2d` (6 bundled
  SKUs) and `pcr300` (4 bundled SKUs) — both still bundle multiple part numbers into one
  card the same way `drivers`/`pcr3d500` used to, but neither is on the whiteboard's
  Active Drivers list, so they were left alone this pass. Worth asking Cory whether "every
  SKU gets its own card" should extend to these too, or whether it's scoped to
  whiteboard-confirmed-active SKUs only.
- Extend the same hub+SKU-card treatment to other categories per Cory's stated long-term
  goal ("the way we built custom strip will now be the gold standard" for every category)
  — Drivers was the second pilot after Custom Strip itself; Lights, Water Features, and
  WiFi/Remotes/Color Touch App haven't been evaluated for the same restructuring yet.

## Front-page navigation grouping (2026-07-09)
Two new top-level hub cards, same `cat-*` hub pattern as `cat-strip`/`cat-drivers` (no
photo, no steps/issues — just an intro line and a "Select a Resource" link list):
- **`cat-troubleshooting`** ("Troubleshooting") nests `framework` (Probability-Based
  Diagnostic Framework), `led-diagnostics` (LED Light Diagnostics), and `morevideos`
  (Other PAL Videos) — all three previously sat as their own top-level cards.
- **`cat-process`** ("Process") nests `quickship` (Quick Ship — Trigger Criteria) and
  `escalation` (Escalation Quick Reference).
All 5 nested cards got `backTo` pointing at their new hub and were added to
`HIDDEN_BY_DEFAULT`; the two new hub ids were not, so they render on the default home
view in place of the 5 cards they absorbed. `cat-troubleshooting` was inserted at the very
front of the `DATA` array (where `framework` used to be first) and `cat-process` right
before `quickship` — front page is now 6 top-level cards instead of 9: Troubleshooting,
Cloning to Competitor Systems (untouched, not part of this request), Products, Strip
Lighting, Drivers & Controllers, Process.
**Confirmed the "Quick Ship"/"Escalation" quick-search chips still work** after hiding
those two cards — the chips just populate the search box and run the normal full-text
search (`searchDATA`), which scans all of `DATA` regardless of `HIDDEN_BY_DEFAULT` (only
the no-search default view filters on that set) — no chip logic needed to change.
Verified in preview: DATA parses (68 entries, no dupes), both hubs expand and list their
resources, all 5 nested cards resolve with a working "← Back to Troubleshooting"/"← Back
to Process" link, both quick-search chips still return results, no console errors.

## Cloning / competitor updates (2026-07-10)
Several small corrections and additions from Cory, all landing on the `competitors` card
unless noted:
- **Jandy panel programming — new content.** The Jandy automation panel's relay/output
  must be programmed as "WaterColors LED" (short code **JL**) for a cloned PAL light to
  get the correct color palette — the similarly-named short code **JC** does not have the
  correct palette and will produce wrong colors even with the PAL driver's DIP/protocol
  set correctly. Added to the Jandy row of the Quick Reference table and as a new
  `issues` entry (a real, common wrong-colors cause that isn't a PAL-side fault).
- **Hayward protocol name corrected to Universal Color Logic (UCL).** Cory confirmed this
  is what current-generation PAL drivers actually label the Hayward protocol menu option
  as — "ColorLogic" (the name previously used throughout this guide) is what it's called
  on older driver menus. Updated everywhere the guide names this protocol setting: the
  `competitorSelect` dropdown label, the `competitors` card's facts/issues/Quick Reference
  table, and the `waterblade` card's LIT driver DIP switch table (the only other DIP
  table in the guide that names protocols rather than just brands). Phrased as "Universal
  Color Logic (UCL) — labeled 'ColorLogic' on older driver menus" rather than a blanket
  replacement, since Cory's framing was specifically about newer drivers — not resolved
  as to exactly when the naming changed over.
- **Pentair IntelliBrite color show/fixed-color reference — new content.** Cory added
  `source-manuals/Competitors/Pentair/globrite-color-changing-led-light-manual...pdf` —
  Pentair's own GloBrite install/user's guide, which documents the 14-item numbered list
  (7 light shows, 5 fixed colors, Hold, Recall) that GloBrite/IntelliBrite lights cycle
  through via wall-switch power-cycling, and which the manual's own text confirms
  IntelliBrite is compatible with/synchronizes to. Added as a new table on the
  `competitors` card under the existing Quick Reference table — useful for translating an
  automation panel's numbered mode/scene into what a PAL light cloned to IntelliBrite
  should actually display. Framed as a translation reference, not a claim that PAL's own
  cloned drivers use the same wall-switch power-cycling mechanism themselves.
- **Search bug — investigated, already fixed, no code change needed.** Cory reported that
  searching a full part number (e.g. "PCR-1Z-65") for the 1Z/1ZW/1Z-SM/2Z/2ZW drivers
  returned nothing, while the shorter "PCR-1Z" worked. Traced this to the *old* bundled
  `drivers` card structure (pre-dating the SKU-first restructuring two sessions ago): its
  title ("PCR-1Z / PCR-2Z / PCR-1Z-SM / PCR-1ZW / PCR-2ZW Drivers") and tags never
  contained a contiguous "1z65"-type substring for the search index's squash-matching to
  find, so full part numbers silently fell through to zero results. Confirmed on the live
  site (rootscx.github.io) that this is already resolved as a side effect of last
  session's SKU-card split — every one of the 5 full part numbers, with or without the
  "64-" prefix, now correctly surfaces its own card as the top result, since each SKU's
  id/title is now the part number itself. Nothing further to do here.
- **DMX hardware-vs-software scope — new content.** Cory's guidance for techs: separate
  what's PAL's responsibility on a DMX call (hardware — lights, cables, and the driver
  being wired/DIP-addressed correctly) from what's the home automation integrator's
  responsibility (programming the DMX controller/automation platform's own software).
  Added as a "Scope of support" note on the `attendant` card (the primary third-party DMX
  integration card, where this boundary matters most) with a full explanation, and as a
  shorter cross-reference line on `pcr2dmx` and `pcr3dmx8z` (the two DMX-capable driver
  cards) pointing back to it — so a tech landing on either the driver-hardware side or the
  automation-integration side gets the same framing.
All five changes verified in preview: DATA parses clean (68 entries, no dupes), each
edited card renders the new content, no console errors.

## Pentair IntelliFlo3/IntelliPro3 VSF pump relay tie-in (2026-07-14)
Cory added `source-manuals/Competitors/Pentair/intelliflo3-pro3-vsf-install-guide.pdf`
(Pentair's own 27-page Installation and Maintenance Guide for the IntelliFlo3®/
IntelliPro3® VSF variable-speed/flow pump) and said PAL drivers "can be mounted directly
to this pump." Read the full manual before building anything — nothing in it shows a
physical mounting bracket/point on the pump body for a separate driver enclosure. The one
real match: the pump's optional Relay Control Board Kit (P/N 356365z, installs inside the
pump's own field-wiring compartment) has a 5A relay terminal (0-300 VAC/0-48 VDC)
explicitly labeled **"Pool Light / Transformer"** in the pump's wiring diagram (manual
p.5), and the pump's own touchscreen/app Relay Settings screen (p.12) offers "Lights" as
a Device Type for that relay. Asked Cory to clarify "mounted directly" against this
finding before writing anything into the guide, rather than guess at a physical-mount
claim on a live troubleshooting card. **Cory confirmed: it's the relay tie-in, not a
physical mount** — team-confirmed a PAL driver's power ties into that 5A relay, the relay
gets programmed as a Pentair color light (IntelliBrite/GloBrite specifically, if that
option is exposed), and as long as the PAL driver's own DIP switches are set correctly
*and* the relay is programmed right, the light should clone properly.
- Added a new "IntelliFlo3/IntelliPro3 VSF Pump — Driver Relay Tie-In (Pentair)" section
  to the `competitors` card, right after the existing IntelliBrite Color Show reference
  table. Framed as an *additional* power/switching path, not a replacement for normal
  cloning steps — both the standard DIP-switch/protocol requirement (already documented
  above it on the same card) and correct relay programming are needed together.
- **Flagged, not overclaimed:** the pump's own manual/touchscreen only expose a generic
  "Lights" device type for the relay — no "IntelliBrite" or "GloBrite"-specific circuit
  option is shown anywhere in this manual. Cory's own phrasing ("if explicitly stated")
  suggested that more specific label may only exist on a full Pentair automation panel
  (IntelliTouch/EasyTouch/IntelliCenter) tied into the same system, not on the pump's own
  screen — worded the card so a tech sets "Lights" at minimum on the pump itself, and the
  more specific color-light type on the automation panel *if* one is present and offers
  it, rather than asserting a specific menu label that isn't confirmed to exist.
- Not gated behind Jason sign-off — this is real install-manual content (a documented
  relay terminal + device-type setting) plus direct team confirmation, not a contested
  diagnostic branch, same bar as the `attendant` (Poolside Tech) card.
- Added tags (`intelliflo3`, `intellipro3`, `vsf pump`, `relay control board`, `pool
  light relay`, `5a relay`) to the `competitors` card for searchability.
- Verified in preview: DATA parses (77 entries), new section renders under Pentair,
  no console errors.
- **Corrected same session, per [[feedback_no_incard_flags]]:** the first pass added a
  trailing amber note-box sourcing the section ("Confirmed by the team and cross-checked
  against the pump's own install manual... not a single documented button/menu sequence
  in either PAL's or Pentair's own materials...") — exactly the build-process/sourcing
  commentary that standing rule says stays out of the guide. Removed. The two `order-note`
  divs above it stay — they're real in-workflow instructions (DIP switches, relay
  programming), not meta-commentary about how the card was built.

## Remotes promoted to front page, restructured SKU-first (2026-07-10)
Third category to get the Custom-Strip-style hub+SKU-card treatment (after Custom Strip
and Drivers), and the first one Cory asked to put at the very top of the front page
rather than in its existing array position.

- **`cat-remotes` inserted as the very first item in `DATA`** — new top-level hub, no
  `backTo`, not in `HIDDEN_BY_DEFAULT`. Home page order is now: Remotes, Troubleshooting,
  Cloning to Competitor Systems, Products, Strip Lighting, Drivers & Controllers, Process.
  Contains a Remote Comparison table (all 6 SKUs: part number, what it is, pairs-with
  driver, active/discontinued status) and a flat "Select a Part Number" list, plus a
  "Shared Reference" section linking the reference card (below) and the existing
  `colortouchapp` card.
- **The old combined `remotes` card was split**, same pattern as `drivers`/`pcr3dref`:
  kept its id, retitled to "Remotes — General Troubleshooting & Pairing Reference," and
  trimmed to hold only genuinely cross-SKU content — the three new notes from Cory (below)
  plus the existing "discontinued vs. spare-parts-only" framing. The six per-SKU sections
  that used to live inside its `extra` (PCZ-2, SR2, PCT-1, PCT-3, PCT-5, SR) each became
  their own leaf card — `pcz2`, `palsr2`, `pct1`, `pct3`, `pct5`, `palsr` — carrying
  forward their existing facts/photos/pairing steps/"Used by" lists unchanged, now in the
  standard photo → facts → steps → issues card shape instead of a `div`-per-section block.
  Each leaf card cross-references the reference card for the three new checks rather than
  repeating them six times.
- **`cat-wifi` retitled** from "WiFi, Remotes, Color Touch App" to "WiFi, Color Touch
  App" and its Remotes link removed, since Remotes no longer nests under it — same
  treatment `cat-strip`/`cat-drivers` got when they were promoted out of `products`.
  Updated the matching link text on the `products` hub too.
- **New remote-troubleshooting content from Cory, added to the `remotes` reference card:**
  1. **Color Wheel / Indicator Check** — first-line diagnostic for any "remote isn't
     working" call: confirm the color wheel (or the white dot in its center, on the
     remotes that have one) is actually lit — hard to see in direct sunlight, covering it
     with a hand helps. If it's not lit, check batteries before assuming anything else —
     the batteries PAL installs at the factory are occasionally bad out of the box.
     Deliberately didn't attribute "white dot" to a specific SKU since nothing on file
     confirms which remotes have the plain-wheel vs. white-dot-center variant — phrased
     generically per Cory's own wording rather than guessing. Also didn't apply this
     wheel-specific step to `pct3` (Commander Touch), whose own existing facts describe
     CH1/CH2 + mode buttons only, no color wheel — flagged explicitly on that card instead
     of overclaiming a wheel exists there.
  2. **Pairing** — every remote/Wi-Fi module should already come paired from the factory;
     don't assume a remote needs re-pairing just because a customer reports it "isn't
     working" — work the Color Wheel/battery check first. If pairing genuinely is needed,
     pull that remote's own card rather than guessing a button sequence. For Wi-Fi drivers,
     the one thing a customer actually has to do themselves is connect to their home Wi-Fi
     network.
  3. **Recommend the Color Touch App** — for a customer needing a new/replacement remote
     on a Wi-Fi-equipped driver, the app (free download, no remote hardware needed) is
     usually the faster fix; a physical replacement remote is still sellable at List price
     if they want one specifically. Added as a `note-box` on the reference card, and
     referenced from each leaf card's own "customer needs a replacement remote" issue row
     — except `palsr2`/`pct1`/`pct3`/`palsr`, whose paired products/drivers don't have a
     Color Touch App path, so those cards say so explicitly instead of pointing to the app.
- **Not resolved, just observed:** the existing `ledbubbler` card's own remote
  troubleshooting text refers to a "red light in center of color wheel," while Cory's new
  note says "white dot" — likely genuine variation across different remotes' indicator
  colors (not a contradiction to fix), but nothing on file maps which SKU shows which
  color. Left both as-is rather than forcing one wording onto the other.
- Updated `productSelect` (6 new SKU options + relabeled reference-card option, in the
  same order as the hub's part-number list) and `HIDDEN_BY_DEFAULT` (added `pcz2`,
  `palsr2`, `pct1`, `pct3`, `pct5`, `palsr`; `remotes` was already hidden).
- Verified in preview: DATA parses (75 entries, no dupes), `cat-remotes` is the first
  `DATA` entry and renders first on the home page, all 8 remote-related cards (hub +
  reference + 6 SKUs) resolve with working back-links, all images 200 OK, all 58
  `productSelect` entries resolve, `cat-wifi` still renders correctly with just WiFi/Color
  Touch App, no console errors.

## Automation / Drivers / Remotes / WiFi document review batch (2026-07-10)
Cory added several new source files across `source-manuals/Automation/`,
`source-manuals/Drivers and Controllers/`, and `source-manuals/WiFi, Color Touch App,
Remotes/` and asked for a full re-review. All 9 new PDFs plus 6 new product photos were
read in full. Built out real content where the new material fills existing gaps or
documents genuinely new equipment; flagged rather than resolved several naming
inconsistencies (a now-familiar pattern in PAL's own source material).

**Gap-fills on existing cards:**
- **`pct1`** — was flagged "no dedicated pairing-steps document on file." A real PCT-1
  manual now confirms the full Matching/Clearing Code procedure (Speed+ within 5 sec /
  hold 3 sec), the 7-mode list (White/Color Change/Disco/4× Gradual Change), memory
  feature, and the 1:unlimited / 4:1 transmitter:receiver relationship — same shape as
  PCZ-2's, different button names. Also newly confirmed: PCT-1's operational LED is
  **red**, not the white dot some other remotes show — a real, product-specific
  difference, not a contradiction to resolve (see the existing `ledbubbler` "red light"
  vs. Cory's "white dot" note from the Remotes restructuring session).
- **`pcr2d`** — was a 2-page-sell-sheet-only card, no install/DIP content. A dedicated
  Bellson Electric (Australia) install/maintenance manual adds real mounting steps, the
  E/A/N + B/G/W/R terminal layout, a max-lights-by-wattage table, and a 2-switch Cloning
  DIP table (no Astral option — simpler than the PCR-1Z/2Z family's tables). Two new
  internal inconsistencies found in this one manual and flagged, not resolved: (1) it
  calls the top wattage tier "55 watt" where this card's existing SKU table (from the
  current US sell sheet) says "60W"; (2) its own wiring diagram labels the output "12V
  20Watt DC" while its spec box says "16 WATTS" — yet another instance of PAL source
  material disagreeing with itself. **Strengthened evidence for the pending 12V/24V
  sign-off question** (see below): this manual is internally consistent at 12V DC
  throughout, independent corroboration alongside the PCR-4 manual (also Bellson,
  also clean 12V DC) — see Pending Sign-off for the updated framing.
- **`pcr4`** — was a photo-only stub ("no install/spec manual has been provided"). Now
  fully built out: mounting/wiring steps, 8-output capacity (up to 8 LAU-4C lamps),
  the same 2-switch Cloning DIP table as PCR-2D, the dry-contact remote on/off connector
  (Part No. 42-PCTWFA kit) separate from cloning and separate from Wi-Fi, and a new flag
  that two different PAL documents show two different Wi-Fi module part numbers fitted
  to this same driver (42-PCTWF1 in one guide, 42-PCTWF5/"Touch 5" in the dedicated PCR-4
  manual) — not resolved, likely different hardware revisions.
- **`pct5`** — was flagged "no pairing/cloning steps document on file." The Touch-5
  Wi-Fi module manual documents PCT-5's real Matching/Clearing Code procedure (different
  button timing than PCT-1's: Code Setting Button + Speed-UP within 2 sec / hold 5 sec).
  Also added a load-bearing clarification: PCT-5 pairs to **two different pieces of
  hardware** that both use "Touch 5" branding — a PCR-4 fitted with the Touch-5 Wi-Fi
  module (a lighting-only retrofit), or a standalone PCR-5CU relay controller (see new
  card below, factory-paired to a PCT-5 out of the box). A tech needs to confirm which
  one a customer actually has before troubleshooting further.
- **`colortouchapp`** — the Touch 5 / Touch 9 coverage-gap flag is now **partially
  resolved for Touch 5**: a dedicated manual confirms it's a Wi-Fi module (42-PCTWF5)
  that retrofits a PCR-4 into a 4-zone-plus-master "TOUCH-5" app profile, matching the
  PCT-5 remote's own layout. Also surfaced a genuinely confusing point worth remembering:
  the same "Touch 5" app icon/branding is shared between that PCR-4 retrofit and the
  entirely separate standalone PCR-5CU 5-channel relay/equipment controller (below) —
  same UI, different underlying hardware. Touch 9 is still completely undocumented.
  New, still-unconfirmed finding: the app's own "PAL Lighting Apps" screen shows a
  fifth icon, **"PCT-3D"**, alongside TOUCH-1/PCT-3/TOUCH-9/TOUCH-5 — unclear whether
  this is a variant of the existing `pct3` Commander Touch remote or a separate product;
  flagged only, not built.
- **`wifi`** — flagged that 64-WIFI (the only module documented on this card) is not the
  only Wi-Fi module PAL makes. New product photos confirm four more part numbers:
  64-PAL-SW ("Sonar Wi-Fi," DMX512 input, Alexa/Google Assistant compatible — used with
  the Sonar remote family), 64-PCTWF03, 42-PCTWF00, and 42-PCTWF5 ("Touch 5," see above).
  Only product photos exist for the first three — no install manuals yet to confirm
  functional differences. This is a real candidate for a future SKU-first split (same
  pattern as Drivers/Remotes) once manuals exist; not attempted this session since there's
  nothing beyond photos to build from for 3 of the 4.

**Two new cards built, both under `cat-automation`** (not promoted to the front page —
`cat-automation` itself stays nested under `products` for now):
- **`pcr5cu`** ("PCR-5CU / PAL Touch 5 — 5-Channel Relay & Light Controller")** — real
  find: this is genuinely not a lighting driver, it's a 5-channel equipment controller.
  3 general-purpose relay channels (CH1-3, for pumps/equipment), 1 channel wired
  specifically for a 120V or 240V 2-speed motor (CH4/CH4A, two relays for low/high
  speed), and 1 standard PAL RGB light channel (CH5) — all from one enclosure,
  controlled via PCT-5 remote and/or the app's TOUCH-5 screen. **Naming flag:** three
  different names found for what appears to be one product across one set of source
  files — folder/SKU "42-PCR-5S" (and "5SW" for Wi-Fi), the manual's own body text calls
  it "PCR-5CU," and the document title says "TOUCH 5 INSTRUCTIONS." Also the (by now
  expected) 12V/24V copy-paste artifact: title says 12V D/C, Important Information says
  24V DC. Built from a real install manual (mounting, wiring diagrams for all 5
  channels) — this is equipment documentation, not ambiguous diagnostic logic, so it
  didn't need the "goes to Jason first" gate; that gate is for contested/ambiguous
  troubleshooting branches, not for onboarding a new product with a clear manual (same
  bar as every other card in this guide). Hero photo: the clean product PNG Cory
  provided (`42-PCR-5S.png`), composited onto white per the standard hero-photo
  treatment, saved as `assets/image_192.jpg`.
- **`pcr2vcu`** ("PCR-2-VCU / PCR-5V — Motorised Valve Control Unit")** — an accessory
  relay box, not a lighting product, that adds motorized 24V pool valve control (up to 3
  valves) to the same Touch 5/9 ecosystem, wired to and powered from a PCR-5CU or
  "PCR-9CU" driver. **Naming flag, worse than PCR-5CU's:** three different names for the
  same physical unit — source folder "PCR-2-VCU," the manual's own body text calls it
  "PCR-VCA," and the actual nameplate pictured on the unit reads "PCR-5V." Also a minor
  IP65 (nameplate) vs. IP55 (manual text) mismatch. Branded "Pool Touch" (fingerprint
  logo) — visibly different branding from "PAL Lighting," worth knowing if a customer
  mentions the name. **PCR-9CU is referenced here and on the `pcr5cu` card as a
  compatible driver but is not documented anywhere in this guide** — presumed to be a
  9-channel counterpart, unconfirmed, no manual on file. Hero photo: `valvecontrol-2.png`
  (two-angle product shot), composited onto white as `assets/image_193.jpg`.
- `cat-automation`'s own intro facts updated to mention PAL's own Pool Touch
  equipment-automation hardware alongside the existing third-party (Poolside Tech)
  integration framing, and both new cards added to its "Select a Product" list,
  `productSelect`, and `HIDDEN_BY_DEFAULT`.

**Not built, flagged as open questions for Cory/Jason:**
1. Whether the "Pool Touch" equipment-automation product line (valve/pump/motor control,
   not lighting) belongs in a guide scoped to "PAL Lighting Tech Support" — built it in
   this pass since the precedent (`attendant`) already accepts automation-adjacent
   content under `cat-automation`, but this is a bigger step in that direction (general
   pool equipment, not just a lighting integration) and worth Cory's explicit sign-off
   that it should stay.
2. The PCR-5S/PCR-5CU and PCR-2-VCU/PCR-VCA/PCR-5V naming inconsistencies above — worth
   asking Cory/PAL directly which name techs should actually use on a call, rather than
   leaving three names live in the guide indefinitely.
3. Touch 9 / PCR-9CU — still no manual on file anywhere. Presumed to be the 9-channel
   counterpart to PCR-5CU/Touch 5, unconfirmed.
4. "PCT-3D" app icon — unconfirmed relationship to the existing PCT-3 remote.
5. Wi-Fi module SKU multiplicity (64-PAL-SW, 64-PCTWF03, 42-PCTWF00, 42-PCTWF1,
   42-PCTWF5) — worth a dedicated SKU-first pass like Drivers/Remotes once install
   manuals exist for the ones that currently only have product photos.

Verified in preview: DATA parses (77 entries, no dupes), all 9 new/edited cards resolve
with working back-links, both new hero images (192/193) load 200 OK, all 60
`productSelect` entries resolve, no console errors.

## Reference-card rename + clickable links, soft reset procedure (2026-07-10)
Cory flagged two mechanical problems with the `drivers`/`pcr3dref` reference cards
introduced during the SKU-first restructuring: (1) every mention of them across the 9
SKU cards and the `cat-drivers` hub was plain bold text, not an actual link — even Cory
couldn't jump to the card from a mention; (2) "Family Reference" was jargon coined this
session, not a name PAL or the guide's own card titles actually use.
- **All 53 prose mentions are now real links.** Added a small `.inline-link` CSS class
  (teal, underlined, `cursor:pointer`) and wrapped every mention in a `<span
  onclick="jumpToProduct(...)">` the same way `back-link`/`product-link` already drive
  navigation elsewhere — tapping any mention now jumps straight to the card. HTML
  attribute quoting uses single quotes + `&quot;` entities for the `jumpToProduct(...)`
  argument specifically so the same replacement text drops safely into both
  backtick-template fields and double-quoted JS string arrays (`steps`/`issues`) without
  clashing with either's own quoting.
- **Renamed away from "Family Reference" everywhere** (dropdown options, the
  `cat-drivers` hub's part-number links, and all 53 inline mentions) — now reads
  "PCR-1Z/2Z Cloning & Pairing Reference card" and "PCR-3D Mounting & Cloning Reference
  card" respectively. The `cat-drivers` hub's own note box ("...lives on their own Family
  Reference cards, linked below") was reworded to "their own reference cards" since it
  refers to both cards collectively and can't carry a single link itself — the actual
  links sit right below it on the two `product-link` divs, already fixed.
- **Soft reset procedure added** to the `drivers` card's pairing steps, right after the
  Matching Code / Clearing Code steps: holding both the Manual Operation button and the
  Code Setting button together for ~10 seconds drains the capacitors immediately, instead
  of powering off and waiting 15-30 minutes for them to discharge on their own before a
  clean reset. New content — wasn't documented anywhere in the guide before.
- Strip Lighting and the SR2 remote (`palsr2`) were explicitly out of scope for this pass
  and were not touched.
- Verified: DATA still parses (77 entries), all replacements land inside valid HTML/JS
  (spot-checked both backtick-field and double-quoted-string contexts).

**Follow-up fix, same session:** Cory caught that the first pass above missed the actual
`title:` fields of both cards — they still read "PCR-1Z / PCR-2Z **Family** — Cloning,
Pairing & Zone Reference" / "PCR-3D **Family** — Mounting, Cloning & Configuration
Reference," which didn't match the renamed dropdown/link text and was exactly the kind
of mismatch this pass was supposed to fix (tap a link, land on a card whose header uses
a name you can't find anywhere else). Fixed both `title:` fields to drop "Family" and
match the link text exactly: "PCR-1Z/2Z Cloning, Pairing & Zone Reference" / "PCR-3D
Mounting, Cloning & Configuration Reference."
- **Also rewrote every generic lowercase "driver family" / standalone "PCR-1Z/2Z family"
  / "PCR-3D family" / "remote family" mention outside Strip Lighting** (~47 "driver
  family" instances plus ~24 standalone "family" mentions) to "driver platform" or a
  case-by-case plain rewrite (e.g. "same general Sonar remote as..." instead of "...remote
  family as...", "one product" instead of "one product family" on the `pcr5cu` naming
  flag) — Cory confirmed he wanted this widened beyond just the capitalized "Family
  Reference" jargon once he saw the same word still causing confusion elsewhere.
  **Caught and reverted one mistake mid-pass:** a first blind find/replace of "driver
  family" → "driver platform" hit 12 lines inside the Custom Strip SKU cards
  (`customstrip-nfwb8` through `customstrip-plo2w`, "Driver compatibility" facts and
  order-notes) before it was caught — Strip Lighting was explicitly out of scope, so
  those 12 lines were reverted back to "driver family" and left untouched. Card ids in
  the file run contiguously by category (Strip Lighting SKUs sit at lines ~929–1535,
  immediately before the `drivers` card at 1536+), which is what made it possible to
  audit "did anything in the strip id-range change" after the fact and catch this.
  `cat-strip`'s own hub facts line ("PAL's Evenglow linear-strip/track family...") and
  every "family" mention inside quickshipstrip/perimeterstripkit/customstrip and its 9
  SKU cards were left alone throughout, per the standing instruction not to touch Strip
  Lighting.
- **Not yet done, flagged during this pass, needs Cory's call:** ~23 cards still contain
  plain-text (non-clickable) mentions of "the Drivers card" and "the PCR-3D-120/500 card"
  by name — both are stale leftovers from *before* this session's SKU-first
  restructuring even started. "Drivers" was the `drivers` card's old title before the
  Family Reference rename (now doubly stale). "PCR-3D-120/500" is worse: that id was
  freed up and reassigned to the actual 64-PCR-3D-500 SKU card during the earlier
  Drivers & Controllers restructuring — the name no longer maps to any single card at
  all (the shared content it used to refer to now lives on `pcr3dref`). This is the
  same class of bug as the one just fixed (stale name, not a real link) — already
  flagged in this file's "Drivers & Controllers SKU-first restructuring" section as a
  known follow-up, not touched yet since it's a larger, separate mass-edit (~23 cards)
  and wasn't part of what was asked for in this pass.
- Re-verified after the revert: DATA still parses (77 entries), Strip Lighting cards
  confirmed unchanged (spot-checked `customstrip-nfwb8` in the live preview — still
  reads "driver family"), no console errors.

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

**New (2026-07-07):** the `ledbubbler` card's own manual has the same "Matching Code"
ambiguity as above, self-contained within one document — Section 6 Troubleshooting's
remote-resync steps (Code Setting once, then Z1 x3) differ from Section 5's own
"Features" list procedure (Z1 once within 5 seconds). Not a new question, just
supporting evidence the ambiguity is real and not a one-off typo — cross-referenced on
the `ledbubbler` card, not re-litigated there.

**New (2026-07-07):** the PCR-3D's "Updates August–September 2025" addendum describes a
newer "3D and 3DMX High Powered" V3 board (SKUs 64-PCR-3D / 64-PCR-3DW / 64-PCR-3DMX,
excluding the 8Z) with 4 dip-switch-selected operating modes (Cloner / Module /
PCR8-legacy / DMX), each with different dip-switch meanings from the standard board
documented in the `pcr3d500` card today. This is structurally new diagnostic content, so
per the standing rule it has **not** been built into the HTML beyond a flag noting it
exists — Jason needs to review the 4-mode logic (mode-select semantics, and whether/how
it should merge with or replace the existing Cloning/Zone tables) before it's built out
as real card content.

**New (2026-07-07):** the new `attendant` card (Automation category — see above) documents
this same V3 "High Powered" board's **DMX mode** specifically, sourced from a third-party
integration guide (Poolside Tech's "The Attendant") that's already live in the field. It's
real-world evidence for the pending V3 review above, not a new question on its own —
flagged on the `attendant` card and cross-referenced here, but not merged into the
`pcr3d500`/`pcr3dmx8z` cards. When Jason reviews the V3 4-mode logic, the DMX-mode dip
table on the `attendant` card is worth checking against whatever PAL's own source says.

**New (2026-07-07), evidence strengthened (2026-07-10):** the `pcr2d` card (built from
PAL's current Color Touch Series 2 sell sheet) states 24V DC consistently for the PCR-2D
driver — but the pre-existing `evenglow` and `evenglownicheless` cards, built earlier
from Evenglow's own install manual, document the same driver as genuinely 12V DC when
paired with PCR-4 for older Evenglow installs. This is not the recurring within-document
copy-paste artifact — it's two different PAL source documents disagreeing about the same
driver's real voltage. Plausible explanation is PAL's known 12V→24V driver evolution (an
older 12V-era PCR-2D vs a newer 24V-era one under the same model name), but nothing on
file confirms that. **2026-07-10 update:** two newly-reviewed dedicated install manuals —
one for PCR-2D, one for PCR-4, both published by Bellson Electric Pty Ltd (Australia) —
are each internally consistent and unambiguous at 12V DC throughout, with no self-
contradiction in either document. This is real independent corroboration that a
12V-DC-era board genuinely exists (not just an artifact), consistent with the Evenglow
cards and consistent with a Bellson-made-Australia vs. current-US-sell-sheet generational
split. Still flagged in all cards, not resolved either direction — Jason should confirm
whether these are actually two hardware generations, and if so around when the changeover
happened and whether "Bellson Electric" vs. current PAL branding tracks that split, so
the guide can tell techs which one they're likely looking at from install date/manufacturer
rather than "check voltage and hope."

**New (2026-07-07):** the `evenglowfiberglass` card's install manual specifies a 2⅜"/2½"
holesaw for the wall-mount hole, but PAL's current sell sheet for the same product
specifies 1⅞" instead — a real diameter mismatch, not a copy-paste artifact (both figures
are stated plainly and consistently within their own document, they just disagree with
each other). Since drilling the wrong size hole is irreversible, this has **not** been
resolved either direction — the card only carries an escalate-level flag telling techs to
confirm against the actual fibreglass nut hardware in hand rather than trusting either
document blindly. Jason should confirm the correct hole diameter for this product before
a tech relies on this card for a live first-time install.

**New (2026-07-08), partially resolved (2026-07-10):** the `colortouchapp` card's
official app guide surfaces two driver types never documented in this guide — "Touch 5"
and "Touch 9" — selectable alongside the familiar 1ZW/2ZW in the app's own driver-setup
screen. **Touch 5 is now built out** (see `pcr4`'s Wi-Fi module section, `pct5`, and the
new `pcr5cu` card) — it turned out to span two genuinely different pieces of hardware
under one app icon: a Wi-Fi-module retrofit of a standard PCR-4 lighting driver, and a
separate standalone 5-channel relay/equipment controller. This was real install-manual
content (mounting, wiring diagrams), not ambiguous diagnostic logic, so it didn't need
to wait on Jason — same bar as any other new product card in this guide. **Touch 9
remains completely undocumented** — no manual has been found for it yet (the
`source-manuals/Automation/Pool Touch 9/` folder exists but only has a product photo,
no install guide) — still flagged only. Also unresolved from the same card: the `wifi`
card's older documented driver-linking procedure (Code Setting + Link button 3x) doesn't
match the current app guide's own on-screen steps (hold Reset 3 seconds, then WiFi
password + Start Configuration in-app) — Jason should confirm whether these are
sequential steps for different scenarios, or the old text is simply stale.

## Handoff / IP considerations (background — not an active task)
Cory needs a clean contract-exit path: PAL should be able to keep updating this guide
after the engagement ends, with no ongoing dependency on Cory's accounts/infra. Current
plan: transfer this GitHub repo to PAL's own org, and PAL links/iframes it from a
HubSpot page. This repo is intentionally simple (no backend, no API keys) so that
handoff is just a repo transfer — keep it that way unless told otherwise.
