---
name: figma-audit
description: "Audit a Figma file or Figma Slides deck for design quality issues: data inconsistencies across pages, type sizing problems, typos, type formatting inconsistencies, alignment of similar elements, jumping pixels (static elements shifting between sequential frames/slides), and design errors. Use when user says 'audit this Figma', 'check my designs', 'find design issues', 'audit my file', 'check for jumping pixels', or invokes /figma-audit."
---

> **Tooling note:** Tool names below (`get_pages`, `scan_text_nodes`, `get_node_info`, `create_sticky`, `set_text_content`, …) are conceptual. With the Figma MCP, do all of them through `use_figma` (Plugin API scripts) plus `get_screenshot` for visual checks. Always `return` data from scripts, escape non-ASCII in returned JSON, and load fonts before editing text. Sticky notes only exist in FigJam/Slides; in design files use a text-note frame or Dev Mode annotations instead.
>
> **Missing fonts:** Brand fonts (custom or licensed display faces) are often unavailable in the MCP runtime. Reading `characters` still works, but editing, `getStyledTextSegments`, and font-property checks fail on nodes where `hasMissingFont` is true. Always read text regardless of font; for edits on missing-font nodes, give the user paste-ready replacement copy instead of attempting the write.

# Figma Design Auditor

Audit a Figma file for design quality issues. Produces a ranked findings report, then offers to fix each issue.

---

## Step 1: Identify the file

If the user provided a Figma URL, use it directly with the Figma MCP tools.
If not, call `get_document_info` on whatever file is currently open and ask:

> "Which Figma file should I audit? Paste a link, or tell me the file name and I'll search for it."

For case-study / portfolio decks, also ask once:

> "Do you have a claims list (the metrics and facts you're standing behind) or a talk track? If you share it, I'll check every number and fact on the slides against it. If not, I'll list every claim I find so you can confirm them."

---

## Step 2: Scope negotiation (do this before scanning anything)

Call `get_pages` and count the pages. Then ask:

**If ≤ 8 pages:**
> "Found [N] pages: [list names]. I'll do a full audit of all of them. Any pages to skip?"

Proceed with the full audit on all pages.

**If 9–20 pages:**
> "Found [N] pages. For a thorough audit I'd recommend focusing — do you want:
> 1. Full audit of all pages (thorough, slower)
> 2. Triage pass first — I'll flag which pages have the most issues, then deep-dive those
> 3. Specific pages only — tell me which ones matter most"

Default to option 2 if the user doesn't specify.

**If > 20 pages:**
> "Found [N] pages — that's a large file. I'll run a smart phased audit:
> **Phase 1** — fast triage across all pages (catches typos, data conflicts, unnamed frames)
> **Phase 2** — deep audit on the pages with the most issues (type, spacing, alignment)
> This gives you broad coverage without overwhelming the report. Sound good?"

Always default to the phased approach for > 20 pages.

---

## Step 3: Phased audit strategy

### Phase 1 — Triage (all pages, always run)

Run these fast, high-signal checks across every page simultaneously:

1. Call `scan_text_nodes` on the full document — collect all text content, font properties, and positions.
2. Call `get_node_info` on each page's root to get top-level frame names and flag obvious structural problems.
3. Copy dump (decks especially): return the full `characters` of every text node longer than ~6 words, grouped by slide/frame, and actually read it as prose. Word-list typo matching is not enough — this is where broken sentences, tense mixing, and wrong labels are caught.

From this data, immediately flag:
- Placeholder / lorem ipsum text on any page
- Unnamed frames ("Frame", "Frame 42", "Rectangle 7") at the top level
- Frames named "Copy", "OLD", "BACKUP", "Unused", "Temp", "Delete me"
- Data label conflicts (same label, different values across pages)
- Broken or ungrammatical sentences in any prose block
- Every metric/number claim (feeds Category H)
- Pages with zero text nodes (likely empty or image-only — note but don't flag as error)

After phase 1, **rank pages by issue density** (issues found ÷ frames on page). Report:

```
PHASE 1 TRIAGE — [File Name]
=============================
Pages scanned: [N]

Hotspots (most issues):
  1. [Page name] — 7 issues found
  2. [Page name] — 4 issues found
  3. [Page name] — 2 issues found

Clean pages (0 issues in triage):
  [Page name], [Page name], ...

Proceeding to deep audit on top [N] pages...
```

### Phase 2 — Deep audit (hotspot pages, or all pages if file is small)

For each page in scope, call `get_node_info` with full detail to get element positions, sizes, fills, and hierarchy. Run all eight audit categories below (A–H). For decks and prototype flows, always run Categories G and H. Cover, overview, and outcome slides always get a full Category C and H read even if triage found nothing — they are the slides reviewers read most closely.

---

## Step 4: Audit categories

### Category A — Data Inconsistencies (Page to Page)

Goal: find the same *type* of data displayed differently on different pages.

**What to check:**
- Same label (e.g., "Total Orders", "Revenue", "Conversion Rate") with different value formats across pages (e.g., "$1,234" on one page vs "1234" on another, or "43%" vs "0.43")
- Same metric or KPI with conflicting values that can't both be true (e.g., "Active Users: 1,200" on a summary page vs "Active Users: 800" on a detail page)
- Date formats that differ across pages ("Jan 15, 2024" vs "1/15/24" vs "2024-01-15")
- Date values that disagree across a sequence — same format but different years or impossible ordering within one mock world (e.g. "Mar 3, 2024" on slide 8 and "Mar 3, 2025" on slide 19 of the same UI). Count years per run of slides and flag the minority.
- Status labels that are inconsistent for the same concept ("Active" / "Live" / "Enabled" used interchangeably)
- Currency formatting inconsistencies ("$1,200" vs "$1200" vs "1,200 USD")
- Percentage inconsistencies ("12%" vs "12.0%" vs "0.12")
- Placeholder data still present on some pages but real data on others ("Lorem ipsum", "Test User", "example@email.com", "00/00/0000", "–––")

**Method:** Group text nodes by their apparent semantic role (using parent frame name + label sibling text as context). Compare values for the same labels across pages.

---

### Category B — Type Sizing

Goal: find font sizes that don't form a consistent, intentional type ramp.

**What to check:**
- Identify all distinct font sizes used in the file. List them from largest to smallest.
- Flag sizes that appear only once or twice — likely one-offs or errors.
- Flag sizes that are close but not equal and appear in structurally similar contexts (e.g., one card title is 16px, another is 15px — likely an accident).
- Flag body text that varies unexpectedly (e.g., most table rows are 14px, but a few cells are 13px or 12px).
- Flag heading hierarchy violations: a heading at a lower level uses a *larger* size than a heading at a higher level on the same page.
- Common valid ramps for reference: 10, 11, 12, 13, 14, 16, 18, 20, 24, 28, 32, 36, 48 — deviations from round numbers are worth flagging.

---

### Category C — Typos and Grammar

Goal: find spelling errors and broken or ungrammatical writing in text layers.

**What to check:**
- Common English misspellings:
  - recieve → receive, seperate → separate, definately → definitely, occurence → occurrence, accomodate → accommodate, occured → occurred, sucessful → successful, accessable → accessible, calender → calendar, priviledge → privilege, restaraunt → restaurant, existance → existence, enviroment → environment, benifit → benefit, consistant → consistent, dependant → dependent (adjective), independant → independent, relevent → relevant
- Missing spaces between words ("TotalOrders", "UpdatedJan")
- Extra spaces ("Order  History" with double space)
- Inconsistent capitalization of the same label across pages ("order history" vs "Order History" — flag if unintentional); mid-sentence capitals on common nouns ("2M+ Customers")
- Truncated placeholder text ("Lorem ipsum", "Heading here", "Description text", "Click here", "Button", "Label")
- Unreplaced variables ("0000", "1234567", "$00.00", "XX%")
- Broken apostrophes or quotes
- Grammar and sentence sense (read every prose block, don't pattern-match):
  - Sentences that don't parse ("amid a huge migrate teams split into camps")
  - Tense switching within a paragraph ("The team was… We need…")
  - Missing punctuation that changes meaning ("like missing refunds saving an estimated…")
  - Run-on stat lines and fragments that read as unfinished
  - Awkward line breaks in headings (a hyphenated word split across lines, a single orphan word)

Do NOT flag intentional technical strings, code snippets, IDs, URLs, or realistic product data.

---

### Category D — Type Formatting Inconsistencies

Goal: find text styled differently in ways that appear unintentional.

**Font weight:**
- Group text by its apparent role (heading, subheading, body, label, caption, button). Flag any node where the weight doesn't match peers in the same role across the file.
- Common issue: one button is `Bold` (700), others are `SemiBold` (600).

**Line height:**
- Flag line height values inconsistent for the same font size and text role.
- Flag `auto` / `0` / missing line height on multi-line text blocks.

**Letter spacing:**
- Flag letter-spacing values that differ for the same font size + role combination.
- Flag positive tracking (letter-spacing > 0) on lowercase body text — usually unintentional.

**Text case:**
- Flag inconsistent use of `UPPER CASE`, `Title Case`, or `Sentence case` for the same element type (e.g., button labels: "Save Changes" vs "CANCEL").

**Text decoration:**
- Flag underlines on non-link text, or missing underlines on text styled as a link.

**Text color:**
- Collect all distinct text colors. Flag colors that are close-but-not-identical to a common token (e.g., `#333333` vs `#353535`).
- Flag white text on a light background or dark text on a dark background.

**Font family:**
- Flag any font family appearing fewer than 3 times — likely a one-off or missing font substitution. Exception: a secondary/display face the user has defined as intentional emphasis (e.g. a display face reserved for overview, highlights, and end slides) — check it appears only where its rule allows.
- Flag two different fonts used for the same apparent UI role.

---

### Category E — Alignment of Similar Elements

Goal: elements that play the same role across frames or pages should snap to the same positions and edges.

**Cross-page alignment (most important):**
Identify repeated structural elements — page headers, nav bars, sidebar widths, action bars, table columns, card grids, modal widths. For each repeated element across pages:
- Compare its x position (left edge) — flag if it differs by more than 2px across pages
- Compare its y position (top edge) — flag if it differs by more than 4px across pages
- Compare its width — flag if it differs when it should be fixed (e.g., a sidebar that's 240px on one page and 244px on another)
- Compare its internal padding — if a card has 16px padding on one page and 12px on another, flag it

**Within-frame alignment:**
Within a single frame, find elements that appear to form a visual group (same type, stacked vertically or arranged horizontally):
- All left edges should share the same x value (±2px)
- All top edges in a row should share the same y value (±2px)
- Gaps between siblings should be consistent — find the mode gap, flag any gap that deviates by more than 2px
- Icons paired with text labels should be vertically centered with the text baseline or midpoint — flag if the icon's vertical center differs from the text's vertical center by more than 2px

**Grid/column alignment:**
- If a page appears to use a column grid, check that frame left edges align to column boundaries. Flag anything that sits between columns.
- Table columns: all cells in a column should share the same x. Flag any cell whose x differs from its column's mode x.

**Label-to-field alignment:**
- Form labels should be consistently left-aligned with their input fields (or consistently right-aligned if that's the pattern). Flag any label that breaks the pattern.
- Key-value pairs ("Status: Active") should have consistent spacing between the key and value across all instances on the page.

**How to detect:**
Use x, y, width, height values from `get_node_info`. Group nodes by apparent role (same parent name pattern, same size, same component name). Compute the mode (most common) x or y for each group. Flag any node whose value deviates from the mode by more than 2px.

**What NOT to flag:**
- Intentional asymmetry (a callout card that deliberately breaks the grid)
- Elements in different layout contexts (a mobile frame vs a desktop frame on the same page)
- Component variants that legitimately differ in size

---

### Category F — Design Errors

Goal: catch structural and visual errors that would fail a design review.

**Spacing inconsistencies:**
- Within a frame, find repeated spacings between siblings. Flag gaps that differ from the mode gap by more than 2px.
- Flag mixed auto-layout and manual positioning within the same frame.

**Clipped / overflowing content:**
- Flag nodes whose bounding box exceeds their parent frame's bounds (`x + width > parent.width` or `y + height > parent.height`).
- Flag nodes positioned at negative x or y within their parent.

**Duplicate / stale frames:**
- Flag frames or groups with identical names at the same level in the layer tree.
- Flag frames named "Copy", "Copy 2", "Frame 37 Copy", "OLD", "BACKUP", "unused", "archive", "temp", "delete me".

**Missing fills / invisible elements:**
- Flag layers with zero opacity or `hidden: true` that are not in a component variant.
- Flag text layers with a fill color identical to their background.

**Placeholder frames:**
- Flag empty frames (no children, no fill, no stroke) outside component definitions.
- Flag frames named "Frame" (default Figma name, never renamed).

**Broken component instances:**
- Note detached instances (components that were detached, now just groups). They won't update with the design system.

**Inconsistent corner radius:**
- Identify the most common corner radius values. Flag elements that deviate (e.g., most cards 8px, one card 6px).

**Color inconsistencies:**
- Collect all fill colors on non-text, non-image elements. Group near-matches (within 5% luminance/hue). Flag near-duplicates.

**Presentation artifacts in mockups (decks):**
- Browser chrome, extension icons, localhost/vercel URLs, oversized cursors, or faint leftover frame outlines around device mockups.

---

### Category G — Jumping Pixels (sequential frame stability)

Goal: when frames are viewed in sequence — a prototype flow, a case-study deck, a before/after pair — anything that is *meant to stay still* must not move, resize, or restyle by even a pixel. Reviewers (hiring managers especially) flip between slides/screens and notice the jump instantly. This is the highest-signal polish check; treat it as its own category, not part of E.

**Define the sequence first:**
- Slides: the presentation order (read the slide grid's rows → children in order; never trust a cached slide list).
- Design files: prototype flow order, or frames that share a name stem (`Checkout - Cart` → `… - Shipping` → `… - Payment`), or left-to-right/top-to-bottom placement within a section.
- Compare each frame with its **neighbours** (N vs N−1 and N+1) and with the **mode** across its run of similar frames. Runs are frames sharing the same chrome — don't compare a mobile frame to a desktop frame or an old-UI run to a new-UI run.

**What must be identical across a run (measure relative to the frame/slide origin):**
- Global chrome: top bar, side nav (x, y, width, **height**), footer, deck header/wordmark, page-number position
- Nav contents: nav items and pinned bottom items (e.g. the Settings gear) — same y
- Page header: title x/y, font size/weight, back-arrow position, primary action buttons' right edge
- Content column: left edge, width, and container padding (e.g. 16 vs 24 vs 96 padding makes the title jump)
- Repeated cards/tables: top edge, width, column x positions and widths, corner radius
- Slide-level elements: title box, body box, mockup frame (x, y, size, radius, shadow), callout labels (x) and leader-line endpoints, logos, captions
- Text style of each static element (font, size, weight, line height, letter spacing, color)

**Tolerance:** 0px is the target. Flag any difference ≥ 1px for chrome, titles, and mockup frames; ≥ 2px for everything else. Sub-pixel diffs (< 0.5px) are fine.

**Method:**
1. For each frame in the run, collect `absoluteBoundingBox` of the static elements relative to the frame origin (walk by layer name + role; fall back to matching by text content and size).
2. Compute the mode for every element property across the run; list every frame that deviates, with the delta (e.g. `Settings gear y 640 → 632, −8px on slide 14`).
3. Also check **overflow that hides a jump**: an element taller/wider than its frame (e.g. nav 800px tall in a 840px frame starting at y 64) will look fine alone but differ when a sibling frame is sized correctly.
4. **Screenshot vs live:** if a frame in the run is a flattened image/screenshot while its neighbours are live layers, rendering (anti-aliasing, font hinting, scale) will jump even if boxes match. Flag it and recommend rebuilding the screenshot live (from the design-system components) at the run's exact geometry.
5. Verify with `get_screenshot` of consecutive frames at the same scale — flipping between them should show zero movement in static regions.

**Fixing (Issue 3 by default; Issue 4 if ≤ 2px on non-chrome):**
- Snap deviating elements to the run's mode values; move containers, not individual children, when a whole column is offset.
- Fix the cause, not the symptom: normalize container padding, reset auto-layout sizing (FILL vs FIXED), correct overflowing heights to `frameHeight − top`.
- When a static element is part of a screenshot, rebuild that region live instead of nudging the image.
- Never mix runs when normalizing (e.g. don't apply the desktop nav height to a 1095px-tall iteration frame, or a new-UI padding to the old-UI screens).
- After fixing, re-measure the full run and screenshot two neighbours to prove zero movement.

**Report format:**
```
[G.1] Settings gear jumps — y 640 on slides 12–13, 632 on 14 (−8px)
      Run: slides 12–17 (mockup frame 1280×800 @ 320,140)
      → Snap to y 640
```

---

### Category H — Claims & Narrative Integrity (case studies, portfolios, pitch decks)

Goal: every number, label, and fact on the slides is true, sourced, correctly labeled, and consistent with every other slide and with the story the presenter will tell. A reviewer who catches one shaky claim discounts every other number in the deck.

1. **Claims ledger.** Extract every metric and factual claim into a table: slide, exact text, number, what it measures. Include percentages, dollar figures, counts ("8+", "2M+", "5M"), team sizes, timelines, and dates.
2. **Source check.**
   - If the user gave a claims list or talk track, match every ledger row to it. Flag any number not on the list, and any number whose value, unit, or wording differs from the list ("~25%" vs "25%", "in beta" dropped).
   - If no list was given, present the ledger and ask the user to confirm each row as measured, projected, or estimated.
   - Flag estimates or derived figures stated as fact ("Est 5M tickets" from a back-of-envelope calculation). Recommend the sourced figure it was derived from instead.
   - Flag projections not labeled as projected.
3. **Label/value semantic match.** The heading must describe what the value actually is:
   - "Revenue generated" over a savings figure
   - "Reach" over a cost figure
   - "Result" over a projection
4. **Cross-slide fact consistency.** The same fact must be the same everywhere:
   - Team composition and roles
   - Project scope and ownership
   - Timelines and dates
   - Company, product, and tool names
   - Mock-world data within one UI story (years, customer names, totals)
5. **Story contradictions.** If a talk track is available, flag slide facts that contradict it (slide says "2 engineers", talk track says "a team of 4"). Also flag framing that works against the case study's own argument (a stat celebrating a fee as revenue in a story about customers being punished by fees).
6. **Attribution.** Flag outcomes that may be attributed to the wrong workstream (a metric from project A shown on project B's slide).

**Severity:** contradictions and unsourced numbers stated as fact are Issue 0 (the user must pick the true value). Mislabels and missing "projected"/"estimated" qualifiers are Issue 1. Formatting drift in the same claim across slides is Issue 3.

**Report format:**
```
[H.1] Unsourced estimate stated as fact — "Est 5M tickets a year" (slide 22)
      Derived from $20M ÷ assumed cost per ticket; not in claims list
      → Replace with "~$20M a year in support cost for one ticket type"

[H.2] Contradiction — slide 22 lists "2 engineers"; talk track says "a team of 4"
      → Decide which is true; suggest "4 engineers (2 joined mid-project)"
```

---

## Step 5: Issue severity categories

Every finding gets exactly one of these four severity numbers. Use the number strictly — do not invent others.

| Level | Label | Meaning | Can I fix it? |
|---|---|---|---|
| **0** | Blocker | Fundamental design or structural problem that requires a designer or PM decision — broken IA, contradictory data that needs a source-of-truth call, unsourced claim stated as fact, missing component, component detached from the design system. I cannot resolve these unilaterally. | ❌ No |
| **1** | UX issue | The design will confuse or mislead users — wrong copy, mislabeled metric, raw slug/placeholder shown to users, state logic gap, empty state missing, interaction ambiguity. I can partially fix copy; structural UX decisions need the designer. | ⚠️ Partial |
| **3** | Fixable (medium) | A real inconsistency or error I can fix directly — typo, broken sentence with an obvious fix, wrong font size, "Latest" vs "Last" terminology drift, frame mislabeled, duplicate UUID, wrong casing, mismatched label or date across frames. Takes a targeted script but is unambiguous. | ✅ Yes |
| **4** | Polish | A tiny, low-risk tidying item — 2px alignment drift, trailing space in a frame name, annotation bubble left inside a frame, single stray one-off. Fixing feels better than not fixing, but nothing breaks if it stays. | ✅ Yes |

---

## Step 6: Build the findings report

```
FIGMA AUDIT REPORT — [File Name]
=================================
Pages audited: [list page names]
Total text nodes scanned: [N]
Claims checked: [N] ([N] confirmed, [N] unsourced)
Audit date: [today's date]

SUMMARY
  ⛔ Issue 0 — Blockers (designer decision required): X
  🔶 Issue 1 — UX issues:                            X
  🔧 Issue 3 — Fixable (medium):                     X
  · Issue 4 — Polish:                                X

─────────────────────────────────────────

⛔ ISSUE 0 — BLOCKERS
  [0.1] Conflicting data — "Active Users" shows 1,200 on Overview, 800 on Analytics
        Page: Overview > Summary Card; Analytics > KPI Row
        → Source-of-truth decision needed before I can fix this

  [0.2] Detached component — Card on Settings page was detached from design system
        Page: Settings > Plan card
        → Re-attach to DS.Card or rebuild from component

  ...

🔶 ISSUE 1 — UX ISSUES
  [1.1] Raw slug shown — "plan_annual_v2" displayed as plan name in subscription table
        Page: Account Details > Membership
        → Needs human-readable label; I can fix the copy once you confirm the mapping

  [1.2] Empty state missing — no results state not designed for the Payment methods tab
        Page: Account Details > Overview
        → Requires a new frame; I can't fix this without a design decision

  ...

🔧 ISSUE 3 — FIXABLE (MEDIUM)
  [3.1] Terminology drift — "Latest order" in card views; "Last order" in dropdown
        Frames: 1204:3381, 1187:2914
        → Standardize to "Last order" — I can fix all instances

  [3.2] Frame names mislabeled — 4 frames named "Customer details / search" show Vendor search UI
        → Rename to "Vendor details / search / ..." — fixable with rename_node

  [3.3] Typo — "Fulfilment date" should be "Fulfillment date"
        Page: Account Details > Orders table header (3 instances)

  ...

· ISSUE 4 — POLISH
  [4.1] Annotation bubble inside frame — design note visible in Quick search frame (1180:1965)
        → Move outside frame bounds or to a separate annotation layer

  [4.2] Trailing space — frame named "Vendor details / search / past results " has trailing space

  [4.3] Icon vertical offset — filter icon is 3px above text label center
        Page: Orders > Filter bar

  ...

─────────────────────────────────────────
PASSING ✓
  ✓ Consistent font family throughout
  ✓ No placeholder/lorem ipsum text found
  ✓ No empty unnamed frames
  ✓ Heading hierarchy consistent across pages
  ✓ No overflowing content detected
  ✓ No jumping pixels across sequential frames
  ✓ All prose blocks read cleanly
  ✓ Every claim sourced and consistent across slides

─────────────────────────────────────────
PATTERNS — what to watch going forward

[One paragraph, 4–6 sentences. Identify the recurring anti-patterns across this audit — not individual bugs, but the underlying habits or process gaps that produced them. For example: if most issues are raw slug values leaking into mock data, say so and suggest a process fix. If frame names are consistently stale because they reflect an old IA, name that. If font sizes keep drifting because section headings aren't using a shared text style, say it. If numbers drift because stats are retyped per slide instead of pulled from one claims list, say that. End with one concrete suggestion the designer can act on immediately to prevent the same class of issues next time. Write this in plain English, addressed to the designer, not as a list.]
```

---

## Step 7: Offer to fix or annotate

After the report, ask:

> "Would you like me to:
> 1. **Fix in Figma** — I'll fix all Issue 3s and 4s directly; Issue 1s I can partially fix (copy only); Issue 0s I'll flag but can't resolve
> 2. **Add annotations** — sticky notes on each issue, color-coded by level
> 3. **Export report** — clean markdown to paste into Confluence or a Google Doc
> 4. Just leave the report here for you to review"

### If fix in Figma:
- **Issue 0**: Add a sticky note describing the decision needed. Do not attempt to fix.
- **Issue 1**: Fix copy/label if the correct value is unambiguous. Flag structural gaps as stickies.
- **Issue 3**: Fix directly — `set_text_content`, `set_font_size`, `set_font_weight`, `rename_node`, `move_node`
- **Issue 4**: Fix directly — same tools, batched to minimize round trips
- **Missing-font text nodes**: don't attempt the write; list paste-ready replacement copy per slide instead.
- Deletions: confirm with user first, then `delete_node`
- For each fix: show what changed, ask "Fixed. Move to next issue?"

### If add annotations:
- `create_sticky` near each flagged element
- Color-code by level: black = Issue 0, orange = Issue 1, yellow = Issue 3, grey = Issue 4
- Label each sticky with the finding ID (e.g., "[3.1] Terminology drift")

### If export report:
- Output full report as clean markdown

---

## Notes

- **Don't flag intentional variation.** Component variants with defined names are probably intentional. Brand-system choices (a defined secondary typeface, botanical imagery, a defined accent color) are intentional — check they follow the user's usage rules rather than flagging them as inconsistency.
- **2px is the alignment tolerance.** Sub-pixel rendering and Figma's own snapping can introduce 1px drift; only flag ≥ 2px deviations.
- **Data in mockups is approximate.** Focus on same-label conflicts and obviously broken values, not every number. Headline claims are not approximate — every stat on a cover, overview, or outcome slide gets a full Category H check.
- **Figma node IDs are your anchors.** Always include page name + parent frame name so the user can navigate directly.
- **Prioritize ruthlessly.** A misaligned page header is High; a misaligned icon in a tooltip is Low. A broken sentence on a cover slide outranks any pixel issue.
- **Phase 1 is never skipped.** Even on small files, triage first so the deep audit is focused and the report is ranked by real signal.
