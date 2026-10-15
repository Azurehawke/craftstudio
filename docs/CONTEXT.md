# CraftStudio: context and plan

**Status:** planning only. No app code yet, on purpose: RevuCraft and CADCraft get refined first,
then this repo becomes the merged app. Written 2026-10-14.

"CraftStudio" is a working title. Pick the final name before any code or packaging uses it.

---

## 1. The idea

One app that does both drafting and plan markup, in the spirit of Affinity's single app with
modes. The toolbars, ribbon and side panels change with what you are doing:

| Mode (working names) | Comes from | Documents | What it is for |
|---|---|---|---|
| **Draft** | CADCraft (Azurehawke/cadcraft) | DXF, DWG | Drawing and editing: command line, object snaps, layers, blocks, dimensions, hatches, layouts and plotting |
| **Markup** | RevuCraft (Azurehawke/pdfcraft, branch `revucraft`) | PDF | Field markup and takeoff: measurements in feet-inches, symbols with labels and counts, layers, photo pins, comments |

Possible later modes: **Takeoff** (counts, lengths and areas across sheets in one table) and
**Field** (a tablet-first layout with big buttons for walking a site).

It must run on Steve's Windows laptop and his Samsung Galaxy Tab S10 FE+ (S Pen, keyboard, mouse),
like the two apps do now.

## 2. The key decision: one shell, two engines

Affinity's modes all edit **one** document format, so switching modes only swaps the tools. Here
the two apps have different document models:

- RevuCraft's document is a **PDF object graph**: pages, annotations, optional content groups
  (PDF layers). Edits are applied as `pdfcraft_engine::Edit`s and saved incrementally.
- CADCraft's document is a **drawing database**: model space, paper-space layouts, entities,
  blocks and CAD layers, with copy-on-write undo, read from and written to DXF and DWG.

A CAD line is not a PDF annotation, and a PDF page is not model space. Merging the models would
be a rewrite. So:

- **One app shell** holds both kinds of documents as tabs. A DXF or DWG tab is in Draft mode, a
  PDF tab in Markup mode, and the chrome changes with the active tab.
- **Both engines stay as they are**, each the library crates of its own repo. The shell depends on
  them; it does not copy or fork their code a second time.
- **The value is in the bridges** between them (section 4), not in pretending they are one format.

## 3. What gets shared once

- Theme: RevuCraft's charcoal dark default, light mode, accent colours (`ui-egui/src/theme.rs` in
  pdfcraft). CADCraft has its own dark palette; the shell picks one set of tokens for both.
- Ribbon and workflow tabs (`ribbon.rs` in pdfcraft), extended with the CAD tool sets.
- One layer panel that looks the same in both modes (pdfcraft `revu.rs` `layers_panel`; CADCraft
  `ui-egui/src/layers.rs`).
- **One symbol library**: placed as custom stamps on a PDF (`pdfcraft_annot::symbols`) and as
  blocks in a drawing. Same ids (`data-jack`, `wireless-ap`, …), same labels, so counts agree.
- Update checks against this repo's releases (pdfcraft `ui-egui/src/updates.rs`, Android
  `apps/pdfcraft-android/src/updates.rs`).
- The Android packaging (pdfcraft `packaging/android/`: no-Gradle APK build, `PdfCraftActivity`
  for picking documents, Drive sync and the camera hand-off). CADCraft has no Android build yet,
  so it gets one through the shell.

## 4. Bridges between the modes

In rough order of value for the effort:

1. **Plot to Markup.** Plot a CAD layout to PDF and open it straight away as a Markup tab. Each CAD
   layer becomes a PDF layer (an OCG) with the same name and colour, so it can be shown and hidden
   while marking up. The drawing scale goes into the page's measurement viewport, so measuring
   works without calibrating.
2. **Layer mapping both ways.** RevuCraft layers are OCGs with `/PCColor` and `/PCLocked`
   (`pdfcraft_annot::layers`); CAD layers have a colour, a lock and on/off. Same names map to each
   other.
3. **One symbol library** (section 3), so a jack placed in either mode is the same symbol and the
   same count.
4. **CAD as a backdrop in Markup.** Open a DXF or DWG in Markup without converting it first:
   CADCraft renders it as the page, and measurements snap to the real CAD geometry instead of to
   paths extracted from a PDF.
5. **Markup back to CAD.** Bring symbols, counts and redlines from a marked-up plan into the
   drawing as blocks and entities on matching layers, so field changes become drawing changes.
6. **One project.** A small project file listing the sheets (PDFs and drawings) of a job, for
   takeoff totals across sheets and for opening the set together.

## 5. How to build it, in stages

Nothing here starts until both apps are refined (section 8).

1. **Shell.** New workspace in this repo that depends on both engines and UI crates (as git
   dependencies or submodules, section 6). Tabs for both document types, mode switching, the
   shared theme, update checks. Each mode behaves exactly like its app does today.
2. **Plot to Markup** and **layer mapping**.
3. **Shared symbol library** and shared layer panel.
4. **CAD backdrop in Markup**, then **markup back to CAD**.
5. **Android**: the shell's APK, using RevuCraft's packaging.

Each stage leaves a working app, so stopping after any stage loses nothing.

## 6. What is in this repo now

- `upstream/pdfcraft`: git submodule, Azurehawke/pdfcraft on branch `revucraft`, pinned at
  `e567c47` (RevuCraft build 14).
- `upstream/cadcraft`: git submodule, Azurehawke/cadcraft on `main`, pinned at `e707ab4`.
- `docs/CONTEXT.md` (this file), `README.md`, `CLAUDE.md` (instructions for agents).

The submodules are there instead of a second copy of the source, so both forks keep their history
and can still take updates from storytold's originals (storytold/pdfcraft, storytold/cadcraft).
Update a pin with `git -C upstream/<name> pull` then commit. Clone with
`git clone --recurse-submodules`.

## 7. Technical notes that will matter

- **Same UI stack.** Both use egui 0.36 and eframe 0.36, with the same split of engine crates under
  a swappable `ui-egui` front end. CADCraft's app is `cadcraft_ui_egui::CadApp`; RevuCraft's is
  `pdfcraft_ui_egui::PdfCraftApp`.
- **Vendored patches.** pdfcraft patches crates.io crates in its workspace (`[patch.crates-io]`:
  hayro, hayro-interpret, hayro-syntax, hayro-jbig2, lopdf, **winit**, **egui-wgpu**; see
  `vendor/README.md`, patch 23 is the hidden-layer markup fix). The merged workspace needs the same
  `[patch]` table, and CADCraft must build against the patched winit and egui-wgpu. Expect to check
  this first in stage 1.
- **Other dependencies.** CADCraft pins `craft-ui` from storytold/craft-libs at a fixed revision and
  reads DWG through `acadrust`. pdfcraft has its own fonts crate and optional `CRAFT_FONTS_DIR`.
  Two crates named `fonts`, `render`, `engine` are fine because the package names differ
  (`pdfcraft-*`, `cadcraft-*`).
- **Two app objects in one window.** Simplest first version: the shell owns one `PdfCraftApp`
  and one `CadApp` and draws whichever matches the active tab. Their chrome (menus, rails, panels)
  then gets unified piece by piece.
- **Android.** Built without Gradle by `packaging/android/build-apk.sh`; app id, label and icon are
  set there. Lessons from RevuCraft: the manifest needs `INTERNET` for update checks; the camera
  is reached with `ACTION_IMAGE_CAPTURE` into a MediaStore entry (no camera permission needed);
  documents from Google Drive go through Android's document picker and are synced back on save.
- **CI.** Claude's workspace can't reach crates.io, so builds are checked on GitHub Actions
  (pdfcraft: `.github/workflows/revucraft-checks.yml` and `revucraft-android.yml`), with logs and
  screenshots pushed to a `ci-logs-*` branch that can be read without a login. Copy that setup.
- **APK size.** RevuCraft's APK is about 25 MB; adding CADCraft makes it larger but it has no
  large models.

## 8. Before this starts: refine the two apps

RevuCraft (see `REVUCRAFT.md` in pdfcraft for the full roadmap):
- Done: feet-inches, calibration presets, Revu-style shell with ribbon and accents, update checks,
  layers, networking symbols with labels and counts, photo pins with the camera.
- Next ideas: electrical, fire alarm and security symbol libraries; angle tool; markups list;
  tool chest; overlay compare; sheet sets; tuning measuring and markup for the S Pen.

CADCraft:
- Get it running and comfortable on the Galaxy Tab (touch and S Pen input, the command line with an
  on-screen keyboard) and on Windows.
- Decide its look so it matches RevuCraft's (charcoal theme, accent colour, ribbon) or the other
  way round, so the merge doesn't have to restyle one of them.

Things that make the merge easier if done in the apps first:
- Keep symbol ids and names identical between the two.
- Keep layer colour and lock semantics compatible.
- Keep each app's UI state inside its app object (no globals), so two can live in one window.

## 9. Ground rules (carried over from both repos)

- Rust only, in the product and the build. Never panic on input: no `unwrap`/`expect`/indexing on
  untrusted data in non-test code, no `unsafe`.
- Clean-room: no code, icons or other assets from Bluebeam, Autodesk, Adobe or Affinity. "Revu",
  "Bluebeam", "AutoCAD" and "Affinity" are their owners' trademarks; this app gets its own name and
  look.
- ArtCraft's names and logos stay out of the forks' branding (they belong to the ArtCraft team).
- Every new asset gets an attribution entry with its licence.
- Licence: MIT OR Apache-2.0, as both upstream projects.

## 10. Open questions

- Final name for the app, and names for the modes.
- One repo with both engines as dependencies (this plan), or eventually moving the shell into one
  of the forks?
- Which look wins: RevuCraft's ribbon or CADCraft's tool sets and command line, per mode?
- Does the project file (bridge 6) belong in stage 1 or later? (2026-10-14: the projects module in
  §11 makes it the job file; likely early.)
- Laser meter (S9, 70 m): does it have a Bluetooth keyboard (HID) mode? Which phone app does it
  use? The app's name usually points to the maker and its Bluetooth protocol.

## 11. Steve's workflow, and what it changes (added 2026-10-14)

How the last job went (an existing building, no drawings supplied):

1. Mapped the floor plan in LibreCAD by typing coordinates and distances (`@0,15`).
2. Added layers for the existing electrical outlets, and for lighting with its switches.
3. Planned where the server/switch rack goes, then the drops.
4. During the install, changes happened on the fly and had to be redrawn in LibreCAD.
5. Wanted, and didn't have: photos of the final placement of equipment, marked up, for the future.

At the end, the drawings go to the job's stakeholders.

**What this changes.** The main document is the CAD drawing Steve makes himself; PDFs come in only
when someone supplies plans. So **Draft mode comes first** and the drawing follows the whole job.
RevuCraft's PDF markup stays for supplied plans.

### The job, start to finish, in the merged app

| Phase | What the app does |
|---|---|
| **Survey** (on site, tablet) | Room-sketch tool built for distances: tap a corner, type `15` and a direction, or `@0,15`; big number pad; walls square by default; rooms close themselves; doors and windows dropped with widths. **Laser meter input** (below) types the number for you. |
| **Existing conditions** | Preset layers (*Existing – Power*, *Existing – Lighting*, *Existing – Switches*, *New – Data*, *New – Equipment*); an electrical symbol set (receptacles, switches, lights, panels) alongside networking; symbols snap to walls and face the right way. |
| **Design** | Place the rack and drops with auto-numbered labels (TIA-606 style patterns); **cable runs** from each drop back to the rack with lengths from the drawing, plus service loop and drops; cable counts and footage before ordering. |
| **Install** | Same file on the tablet; moving a drop keeps where it was planned and where it went; each item goes **Planned → Installed → Tested**; an Install view shows what's left with big buttons. |
| **Photos** | Take a photo from a drop or the rack and it is pinned there in the drawing; **mark up the photo itself** (arrows, circles, labels, a ruler calibrated from a known size); tapping a symbol shows its photos, status and cable run. |
| **Handover** | One command makes the stakeholder package (below). |

### Laser distance meter

Steve's meter: an "S9 229ft/70m Laser Measuring Tool with Phone App" (Amazon, no-name brand; the
maker and its phone app's name are still to find out). Two ways in, cheapest first:

1. **Keyboard mode.** Many of these meters can pair as a Bluetooth keyboard and "type" each
   reading. If his can, the survey tool works with it on day one: the reading lands in the distance
   field like typed digits. Check the manual or the meter's Bluetooth menu for an HID/keyboard
   option.
2. **Its own protocol.** Otherwise the meter speaks Bluetooth Low Energy with a vendor protocol,
   often a simple serial-style service sending the reading as text. Find it by connecting with a
   BLE explorer app (e.g. nRF Connect) and taking a reading; then write a small reader for it
   (Android through JNI; Windows through its Bluetooth API). Support a list of known meters and
   add more as they are found.

### Handover package for stakeholders

- As-built drawings: PDF (one sheet per floor, title block, legend, revision) and DWG/DXF.
- Drop and cable schedule (drop ID, room, rack/panel/port, cable type, length, test result).
- Counts by symbol and layer; equipment list with rack elevation.
- Photo report by room or drop, with the marked-up photos.
- Optional: the job file itself, read-only, so the next contractor has everything.

### Money: an estimating and job-costing module

Built on the takeoff, so the numbers come from the drawing rather than being typed twice:

- **Price list**: parts with cost and sell price (cable per foot, jacks, plates, patch panels,
  racks…), and **labor units** (minutes per drop, per termination, per rack).
- **Estimate / quote**: counts and cable lengths × prices, labor hours × rate, markup, tax,
  contingency → a proposal PDF for the customer.
- **Change orders**: drops added, moved or removed in the field already show in the drawing; the
  difference from the quoted plan becomes a priced change order to send for approval.
- **Budget vs actual**: purchases and hours logged against the job; margin as the job goes.
- **Invoices**: progress or final, from the estimate and approved change orders.
- Keep accounting out: export to CSV (and QuickBooks-friendly formats) rather than becoming an
  accounting program.

### Projects: a light project-management module

- A job holds its drawings, PDFs, photos, estimate, schedule and contacts (stakeholders).
- Phases and tasks tied to the drawing: "terminate 2nd floor" knows its drops and their status, so
  progress fills in by itself as items are marked Installed and Tested.
- Schedule (dates and a simple timeline), punch list, daily log with photos, time tracking per
  task, customer sign-off on the handover.
- Works offline on the tablet; syncs through Drive.

### Revised order of work

1. CADCraft on the tablet: distance entry and the survey tool; symbols as blocks (shared library
   with RevuCraft); preset layers; photos pinned to a drawing.
2. Statuses (Planned/Installed/Tested) and the Install view; photo markup.
3. Cable runs and the drop/cable schedule; the handover package.
4. Laser meter (keyboard mode first, then BLE).
5. Estimating and change orders; then budget vs actual and invoices.
6. Projects module.
7. The merged shell (section 5) can come at any point after 1; the modules above live in the
   engines, so they don't wait for it.

## 12. LiDAR room scan, and a home-inspection module (added 2026-10-14)

### LiDAR room scan (a third survey method)

Next to typed distances and the laser meter: a **spinning 2D LiDAR** such as the Slamtec RPLIDAR C1
(360°, about 12 m range, 1–3 cm accuracy, USB serial; Slamtec publishes the protocol).

- The tablet has no LiDAR of its own and doesn't need one: the sensor plugs into its USB-C port
  through an OTG adapter, is powered by it, and is read through Android's USB host API (a USB
  serial bridge; no driver needed). Windows reads it as a serial port.
- On a tripod at waist height, one scan traces the walls of a room (or half of one). The app fits
  straight wall lines, squares the corners, and places the room in the drawing at real scale;
  rooms are joined at a shared doorway or wall.
- LiDAR gives the shape, the laser meter the dimensions that must be exact.
- Optional cordless version: an ESP32 board and a USB battery under the sensor, sending readings
  over Wi-Fi or Bluetooth.
- Bright sunlight degrades these sensors; outdoors, see the lot notes below.

### Home inspections (Steve is considering them as a side business)

Map the house and lot quickly, pin photos and notes to the plan, and produce the customer report.

**On site**
- Floor plan by LiDAR scan and laser meter (above); one sheet per level.
- **Lot**: outline from the county parcel map or an aerial image imported and scaled, then adjusted
  with laser measurements; the tablet's GPS (a few metres) is only good for rough placement.
  Structures, driveway, grading and drainage drawn on top.
- **Findings**: tap the plan where the issue is, take photos (the camera hand-off already works),
  mark them up, pick a category and a rating, and add notes. Dictation would help in crawlspaces
  and attics.
- **Categories** by building system (roof, exterior, grounds and drainage, structure and
  foundation, electrical, plumbing, HVAC, interior, attic and insulation, appliances…) and
  **ratings** (safety hazard, repair, monitor, maintenance, informational).
- **Comment library**: saved narratives per category ("Missing GFCI protection at…"), inserted and
  edited, so common findings take seconds.
- Checklist per system so nothing in the inspection scope is skipped.

**The report** (PDF, generated by the PdfCraft side)
- Cover, property and client details, scope and limitations, a summary of findings by rating.
- The **whole layout** of each level and the lot, with numbered markers for every finding.
- For each finding: a **slice of the plan** around the spot (a zoomed crop with the marker), a
  callout with the marked-up photo(s), the category, rating and notes.
- Findings grouped by system, with page links from the markers and the summary.
- Optional: an interactive PDF where tapping a marker jumps to its finding.

**Business side** (reuses §11 money and projects): the inspection is a job with the client,
address, fee, appointment and invoice; report templates with Steve's branding.

**To check before relying on it**: the state's licensing rules and the Standards of Practice that
apply (state rules, InterNACHI or ASHI) decide what the report must cover; the checklist and
report template should follow them.
