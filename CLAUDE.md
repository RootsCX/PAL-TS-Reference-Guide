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
  independently in the Evenglow Nicheless, Treo Max+, Treo Retro, and Treo Mini+ source
  manuals — each has a troubleshooting-table row or install-diagram callout saying
  "12volts DC"/"12 VAC" that contradicts 24V/24VAC stated repeatedly elsewhere in the same
  document (and in Treo Retro's case, the driver page also mislabels the product as "the
  PAL Treo Max"). This is a stale/reused table template, not a real spec difference — treat
  24V DC as correct and flag the discrepancy in-card rather than rewriting the source
  manual's wording.
- Water intrusion is the most common root cause of shorts in flashing-light scenarios.

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

## Pending sign-off
Decision-tree diagrams (Master Triage, Driver Power and Manual Test, Cloning and DIP
Switch Check, White/Primary Color Test) were sent to Jason as a standalone PDF for review.
Two explicit judgment calls are flagged for him:
1. Whether the cloning check always precedes the color test.
2. Whether a failed white-mode test loops back to driver internals or goes straight to
   replacement.
**Do not build these diagrams into the HTML until Jason has signed off.**

## Handoff / IP considerations (background — not an active task)
Cory needs a clean contract-exit path: PAL should be able to keep updating this guide
after the engagement ends, with no ongoing dependency on Cory's accounts/infra. Current
plan: transfer this GitHub repo to PAL's own org, and PAL links/iframes it from a
HubSpot page. This repo is intentionally simple (no backend, no API keys) so that
handoff is just a repo transfer — keep it that way unless told otherwise.
