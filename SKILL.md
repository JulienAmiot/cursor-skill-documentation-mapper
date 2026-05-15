---
name: documentation-mapper
description: >-
  Gathers documentation from one or more sources (primarily Confluence, but the
  operator may pick others) and maps it into a new document built from a
  template (typically a Confluence page or template). For every mapped block it
  adds a source-URL footnote, marks empty target sections as "content missing"
  without overemphasising them, routes source content that does not fit into a
  dedicated "Discrepancies" section, and ends with a coverage footnote that
  quantifies BOTH missing% (empty target sections / total target sections) AND
  excess% (source characters that did not make it into the final doc, per topic
  and overall). Two modes are offered upfront: SINGLE produces one document
  from one template; SET produces several complementary documents from N
  templates against the same source, then publishes an overview/coverage page
  that cross-references the drafts and reports aggregate ownership coverage
  plus a reference-map / dedup-ratio summary (in set mode every source block
  has exactly one owner draft; the owner is the EARLIEST-RANKED template
  whose best section scores >= threshold for that block — parents are the
  source of truth, even when a child draft would semantically be a better
  fit — so the reference graph is strictly backward (later rank -> earlier
  rank) and acyclic; other drafts emit excerpt-include macros or plain
  backward smart links pointing at the owner rather than duplicating the
  body, and parent drafts never reference children). Use when the user
  asks to map, migrate, consolidate,
  transcribe or rewrite documentation into a Confluence template, mentions
  "documentation mapper", "doc mapping", "fill a Confluence template from
  existing docs", "documentation set", "document a feature end to end", or
  wants a Confluence-to-Confluence proof of concept of template-driven
  documentation. Also use when the operator describes the set-mode goal in
  de-duplication terms — "deduplicate documentation", "dedupe my docs", "no
  duplication across pages", "single source of truth across docs", "make
  these docs reference each other", "cross-reference instead of copy",
  "excerpt-include between pages", "one owner per piece of information",
  "hierarchy of documentation with references not copies", "auto-rank
  templates by title index", "reading order across a documentation set",
  "parent is the source of truth", "no parent-to-child references",
  "remove cycling references in the doc set", "backward references only".
  When the destination supports labels/tags/metadata, also tags
  every published page with the feature name, the per-template kind label
  ("technical", "functional", or "overview" for the SET-mode index page), and
  "cursor".
---

# Documentation Mapper

Maps documentation from arbitrary sources into a target template, footnotes
every block with its source URL, quantifies what got lost (empty target
sections) and what got rerouted (source content that did not fit anywhere in
the template), and tags the destination so it can be found again.

The default proof of concept is **Confluence → Confluence**.

The skill runs in one of two modes, picked upfront in Step 0:

- **`single`** (default) — one source set, one template, one published doc.
  Steps 1 → 11 below.
- **`set`** — one source set, N templates, N published docs PLUS one
  timestamped overview/coverage summary that cross-references them and
  reports aggregate ownership coverage plus a reference-map / dedup-ratio
  summary. In set mode every source block has exactly one owner draft
  (best-fit semantic match); other drafts emit `excerpt-include` macros
  pointing at the owner rather than duplicating the body. The summary
  can be written locally (markdown) or published to Confluence — the
  operator picks in Step 12.0. Steps 1 → 12 below; Step 3 runs once
  (shared source set), Step 4 runs once across all N templates (so the
  cross-product scoring in Step 5 has visibility), Steps 5–11 run once
  per template against the same `placements` map.

## Workflow

Copy this checklist and update it as you go:

```
- [ ] Step 0: Choose run mode (single or set)
- [ ] Step 1: Gather operator inputs
        (set mode: also detect reading order from template titles
         and have the operator confirm)
- [ ] Step 2: Discover the Atlassian MCP server in the current project
- [ ] Step 3: Fetch and split the source(s) into blocks      (shared in set mode)
- [ ] Step 4: Fetch and parse the target template structure  (per template in set mode)
- [ ] Step 5: Map each source block to a target section      (per template in set mode)
- [ ] Step 6: Add a source-URL footnote to every mapped block
- [ ] Step 7: Mark empty target sections as "content missing"
- [ ] Step 8: Build the "Discrepancies" section from excess content
- [ ] Step 9: Append the coverage footnote (missing% + excess%)
- [ ] Step 10: Publish or preview at the destination
- [ ] Step 11: Tag the destination (labels / tags / metadata) when supported
- [ ] Step 12: SET mode only — produce the timestamped overview/coverage summary
        (ask: local markdown / Confluence / both)
```

## Step 0 — Choose run mode

Ask the operator **before** any other question:

```
"Run mode?"
  - single        — one template → one document (default)
  - set           — N templates → N documents + an overview/coverage page
                    cross-referencing them
```

Defaults:

- If the operator's request mentions a single doc kind ("write the runbook",
  "fill the ADR template"), default to `single` and confirm.
- If the operator's request implies covering a feature from several angles
  ("document the X feature", "produce the full doc set for X", "functional
  AND technical doc"), default to `set` and confirm.

Set-mode rules carried through the rest of the workflow:

- Step 1 asks for **N templates** (not one) and a per-template kind label.
- Steps 3, 5–11 run **once per template** against the same Step 1 / Step 3
  source set. Source fetching in Step 3 happens **once and is cached** —
  re-fetching the same pages N times is wasteful and risks version drift.
- Step 12 fires after the last per-template run completes.

See [reference.md § Documentation set mode](reference.md#documentation-set-mode)
for the AskQuestion block, the de-duplication-by-reference algorithm, the
ownership-aware per-block tracking schema, the aggregate-coverage and
reference-map formulas, and the overview page template.

## Step 1 — Gather operator inputs

Use the `AskQuestion` tool when available, otherwise ask conversationally.
Always ask, even when context suggests an obvious answer.

**Inputs gathered once for the whole run** (single AND set mode):

1. **Source type** — default `Confluence`. Offer at least: `Confluence`,
   `Jira`, `Markdown / local files`, `PDF`, `Web URLs`, `Mixed`.
2. **Source location** — for Confluence, accept any of:
   - one or more page IDs,
   - a parent page ID (descendants will be pulled),
   - a CQL query (e.g. `space = ENG AND label = onboarding`),
   - a space key (use with care — can be huge).
3. **Destination** — either new page(s) (just the `parent page ID`; the title
   is inferred in Step 10, not asked here) or an existing page to update.
   Always confirm the destination space. In set mode, all N drafts AND the
   overview page land under the same parent.
4. **Feature name** — short human name of the feature being documented.
   Used as a label in Step 11 (slugified: lower-case, spaces → `-`,
   non-alphanumerics dropped, max 50 chars). If the operator's intent makes
   it unambiguous (e.g. "document the auth feature"), propose a default and
   ask them to confirm rather than asking from scratch.

**Inputs gathered per document** (single mode: 1 entry; set mode: N entries):

5. **Template location and selection** — ask **explicitly** before doing
   any scanning. Never silently pick templates by guessing where they
   live; the operator decides the source. Use `AskQuestion` (single
   blocking prompt) with these three options:

   - **Folder or parent page** — operator provides one Confluence
     folder or page ID; in set mode every direct child (templates) is
     a candidate. The skill then expands the children with
     `getConfluencePageDescendants` (or `searchConfluenceUsingCql` for
     a folder), and in set mode presents the resulting list back to
     the operator for multi-select. In single mode, if the folder
     contains exactly one child, use it; otherwise the operator picks
     one child explicitly from the list.
   - **List of distinct page/template IDs** — operator pastes one or
     more Confluence page IDs (or template IDs) explicitly. No
     expansion. In set mode, the order in which the operator listed
     them is captured as a fallback reading-order (used only if Step
     1.5's index-detection cannot infer a hierarchy — see § Set-mode
     only — detect and confirm the reading order below).
   - **(Set mode only) Auto-scan the destination space** — convenience
     path: scan the destination space for pages carrying a `template`
     label or matching `*template*` / `<feature name>*` titles
     (CQL: `space = <KEY> AND label = "template"`, plus a title-LIKE
     fallback), present the results as a multi-select list. Skip this
     option in single mode (auto-picking a single template from a
     whole space is too risky).

   Always echo the resolved template list back to the operator
   (`Title · Page ID · top-level section preview`) and let them
   confirm or change the selection before continuing. The structure
   (heading tree) AND any example content under each heading are both
   inputs to the mapper.

6. **Doc kind** — `technical` or `functional`. Used as a label in Step 11.
   In set mode, ask once per template; default from the template title
   (`functional documentation` → `functional`, `runbook|operations|design|
   architecture|deployment|testing` → `technical`) and let the operator
   override.

### Set-mode only — detect and confirm the reading order

After the templates are picked, **derive a reading order from a leading
alpha-numerical index in each template title** (e.g. `1. Functional`,
`2.1 Data flow`, `[3] ADR`, `A. Overview`, `B1) Runbook`). The detection
rule, sort key, and failure modes are spelled out in
[reference.md § Detecting the reading order from template titles](reference.md#detecting-the-reading-order-from-template-titles).
Summary:

- Detect a leading index per template using the regex in reference.md.
- Sort by **natural sort** on the index segments (so `2.10` > `2.2` > `2.1`).
- **Mixed schemes** (numeric and alpha in the same set) are **tolerated
  but flagged**: numeric sorts before alpha at the same depth; surface a
  warning AND treat the hierarchy as inferrable only if no other failure
  applies.
- **Duplicate indices** across two or more templates (e.g. `1` and `01`)
  are a **hard failure** of detection — the skill cannot pick an order
  by itself.
- **Mixed indexed/unindexed templates** (some templates have a leading
  index, others don't) are a **hard failure** — the skill cannot place
  the unindexed ones reliably.
- **No detectable index on any template** is a **hard failure** — there
  is no signal to sort on at all.

#### Decision branch — confirm vs. ask

After detection, branch on whether the hierarchy is **inferrable** (every
template has a unique, non-ambiguous index) or **not**:

**Inferrable (no hard failures).** Print the consolidated plan as a
table **sorted by the detected reading order**:

```
# · Rank index · Template title              · Doc kind   · Inferred title
1 · 1          · 1. Functional Documentation · functional · <feature> - Functional Documentation
2 · 2          · 2. Architecture             · technical  · <feature> - Architecture
3 · 2.1        · 2.1 Data flow               · technical  · <feature> - Data flow
```

Print any mixed-schemes warning **above** the table. Then ask **one
blocking confirmation** with three options: `Confirm`, `Re-rank
manually` (operator restates the order), `Cancel`. Do not start Step 2
until the operator confirms.

**Not inferrable (any hard failure).** Do NOT fall back to a default
order with `?` placeholders. Instead, block the run with an **explicit
operator prompt**:

```
"I could not infer the reading order from the template titles.
 Reason: <no leading index found | duplicate indices: <list> | mixed
 indexed/unindexed templates: <list>>.
 Please specify the reading order explicitly: list the templates from
 first to last (rank 1 → rank N)."
```

Use `AskQuestion` with an open-text answer (one option per template,
operator picks them in order — or a free-text field accepting the
operator's numbering). Loop until every template has a unique rank in
1..N. Only then proceed to Step 2. Cancel is always available.

When the operator entered the templates in Step 1 input 5 as a **list
of distinct page/template IDs**, propose that list order as the default
in the prompt above — but still surface the failure reason so the
operator knows the index-detection did not auto-rank them, and require
explicit confirmation before proceeding.

Single mode (N = 1) skips this sub-step — the index is still detected and
stripped from the inferred title (Step 10), but there is no order to confirm.

Record every answer back to the operator before doing any work.

## Step 2 — Discover the Atlassian MCP server and its capabilities

### 2.1 — Server discovery

The Atlassian MCP server identifier varies per workspace. Discover it
dynamically before calling any tool — never hard-code a server id in the
skill output. From the workspace root:

```
ls mcps/*/tools/getConfluencePage.json
```

The parent directory of the match is the server id. Observed patterns:
`user-atlassian` (user-scoped MCP), `project-<slug>-atlassian` (project-
scoped MCP). Use the discovered id in every `CallMcpTool` call. If no
match is found, tell the operator the project has no Atlassian MCP
configured and stop.

### 2.2 — Capability probe

The publish-side tools the skill relies on are NOT all guaranteed to be
present on every Atlassian MCP. Probe the discovered server and record
availability before continuing. See
[reference.md § Destination MCP capability check](reference.md#destination-mcp-capability-check)
for the full check list. Minimum checks:

| Tool                    | Required?                                | If missing                                                                                                  |
|-------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| `createConfluencePage`  | always                                   | Stop — no path to publish.                                                                                  |
| `updateConfluencePage`  | only when Step 1.3 chose "update existing page" | **Stop and explain** — operator must switch to "new page(s) under <parent>" or cancel. |
| any `*Label*` tool      | optional                                 | Note it; Step 11 will take the visible-body fallback.                                                       |

Run the probe with:

```
ls mcps/<server>/tools/createConfluencePage.json
ls mcps/<server>/tools/updateConfluencePage.json
ls mcps/<server>/tools/ | findstr /I label   # POSIX: grep -i label
```

Echo a one-line capability summary back to the operator before Step 3:

```
Capabilities: createConfluencePage [ok] · updateConfluencePage [missing]
              · label tool [missing — Step 11 will use the body fallback]
```

If the operator picked "update existing page" in Step 1.3 and
`updateConfluencePage` is missing, stop here with a short error message
that points them at the two viable next moves: switch the destination to
a new page under a parent, or cancel.

## Step 3 — Fetch and split the source(s)

For each Confluence source, call:

- `getConfluencePage` to read content + canonical URL,
- `getConfluencePageDescendants` if the operator pointed at a parent page,
- `searchConfluenceUsingCql` for query-based sources,
- `getPagesInConfluenceSpace` only if a whole space was requested.

Split each page into **blocks**. A block is the smallest unit you would footnote
on its own — typically a heading + its body, a standalone list, a table, or a
fenced code sample. Keep this metadata per block:

```
{
  "source_page_id":  "...",
  "source_url":      "https://.../wiki/spaces/.../pages/...",
  "source_heading":  "Onboarding > Provisioning",
  "char_count":      <int>,        # used by the excess metric
  "topic":           "<short tag>", # see Step 5
  "raw":             "<storage-format snippet>"
}
```

For non-Confluence sources, do the equivalent: fetch, split into blocks, keep
`source_url` and `char_count`. If a source has no stable URL (local file),
record an unambiguous reference like `file:///<absolute path>#<line range>`.

## Step 4 — Fetch and parse the target template(s)

Call `getConfluencePage` on the template id. Build the **target tree** for
each template:

```
[
  { "path": "1. Overview",                   "examples": "<text>" },
  { "path": "2. Architecture",               "examples": "<text>" },
  { "path": "2.1 Data flow",                 "examples": "<text>" },
  { "path": "3. Operations > 3.1 Runbooks",  "examples": "<text>" },
  ...
]
```

Both the heading text AND any example content under each heading are signals
for matching in Step 5. Preserve the original heading order — the final
document MUST follow the template order.

**Set mode**: parse **all N templates** before moving on to Step 5. Step 5's
cross-product scoring needs every section in every template visible at the
same time so it can pick the single best-fit owner per source block. Do not
interleave Step 5 with Step 4 in set mode.

## Step 5 — Map each source block to a target section

For every source block, score every target section on:

- semantic similarity of the block's heading to the target heading,
- semantic similarity of the block's body to the target's example content.

### Single mode

Score the block against every section of the one template. Pick the
highest-scoring section, breaking ties by depth (deepest wins — specific
sections beat generic parents).

If the best score is below a confidence threshold, **do not force the
match**. Mark the block as `excess` and tag it with a short `topic`
derived from its heading. Excess blocks go to the Discrepancies section
in Step 8.

### Set mode — single owner + cross-references

Set mode applies the **de-duplication-by-reference** rule: each source
block has at most ONE owner across the whole set, and every other draft
that wants the same block emits a cross-reference macro (Step 6) instead
of copying the body. The full algorithm, rendering, and bookkeeping
schema live in
[reference.md § Set-mode de-duplication via reading-order references](reference.md#set-mode-de-duplication-via-reading-order-references).

**Ownership is parent-priority**: a block's owner is the **earliest-ranked
template** (lowest rank index from Step 1.5) that has a section scoring
≥ threshold for that block — even if a later-ranked (child) draft has a
strictly higher score. This guarantees the reference graph is strictly
backward (child → parent), is acyclic, and contains zero parent → child
references.

Summary of what Step 5 does in set mode:

1. **Score the block against every (draft, section) pair** across all
   templates parsed in Step 4 — not draft-by-draft. The cross-product is
   `|blocks| × Σ|sections_of(draft)|` evaluations per block.
2. **Walk drafts in reading-order rank, low → high.** For each draft,
   pick its best-fit section. The **first draft whose best section
   scores ≥ threshold becomes the owner**, with that section as
   `owner_section`. Ties within a draft are broken by section depth
   (deeper wins).
3. **Every later-ranked (child) draft** whose own best-fit section also
   scores ≥ threshold gets a placement `referenced`. References are
   emitted in Step 6 as `excerpt-include` macros (when supported) or
   plain backward smart links (fallback); the body lives exclusively on
   the owner page. Earlier-ranked drafts (lower or equal rank to the
   owner) NEVER emit references — by construction, no parent draft
   points at a child.
4. **If no draft scores ≥ threshold for this block**:
   - Mark the placement `excess` for the **best-fit-below-threshold**
     draft only. The block goes to that one draft's Discrepancies
     section in Step 8 — never duplicated across multiple Discrepancies.

Consequences of parent-priority that the operator should expect:

- The rank-1 (parent) draft accumulates **most of the content** when its
  template covers a broad range of section types — it is the
  source-of-truth document for the whole set.
- Later-ranked drafts become **reading-order-specific views** of the
  same content: many of their sections are reference-only (just a smart
  link or `excerpt-include` to the parent). High `reference%` on these
  drafts is expected and healthy under this rule.
- Section depth and template wording matter more than under best-fit
  ownership — if a child template has the **only** matching section for
  a block, the child still owns it (the parent simply has no fit). This
  is what keeps the child drafts from being purely empty.

Update the shared `placements` map (per-block tracking schema in
reference.md) as you go. Steps 6, 8, 9, and 12 all read from it.

## Step 6 — Source-URL footnotes (owners) and excerpt-include macros (references)

**Owned blocks (single mode AND set-mode owners)** MUST carry a footnote
pointing at their `source_url`. In Confluence storage format, prefer the
footnote macro when available; otherwise use inline superscript markers
(`[1]`, `[2]`, …) and a `References` block at the end of each section
listing the URLs in order. Never silently merge two blocks from different
sources without preserving both URLs as distinct footnotes.

**Set-mode owners** additionally wrap each owned section's body in a
**named excerpt** macro so other drafts can include it:

```html
<ac:structured-macro ac:name="excerpt">
  <ac:parameter ac:name="name">{owner_excerpt}</ac:parameter>
  <ac:rich-text-body>
    {body with source-URL footnote}
  </ac:rich-text-body>
</ac:structured-macro>
```

where `{owner_excerpt}` is `slugify(owner_section)` — see reference.md.

**Set-mode references** (blocks marked `referenced` in Step 5) do **not**
carry a source-URL footnote (the owner already does) and do **not** copy
the body. They emit one short italic introduction plus a backward
pointer to the owner. The pointer renders in one of two equivalent
ways — pick the first that the destination supports, both are
acceptable:

1. **`excerpt-include` macro** — preferred when the destination
   Confluence supports named excerpts. The owner wraps each owned
   section in a named `excerpt` macro (see snippet above); the child
   draft pulls it in with `excerpt-include`. Render is inline; the
   reader sees the actual owner body in context.
2. **Plain backward smart link** — fallback when named excerpts are not
   available, or when the operator prefers a shorter visual footprint.
   The child emits an `<ac:link>` (or a `<a data-card-appearance="inline">`
   smart-link tile, in the modern editor) pointing at the owner page
   and section. The reader clicks through instead of reading inline.

Because ownership is parent-priority (Step 5), references are strictly
**backward** in the reading order — a child draft pointing at its
parent. Owners are therefore **always published before** the drafts that
reference them (Step 10 publishes in reading-order rank), so the
pointer always resolves at the moment the child page is created. Both
title-based (`SPACE:<destination title>`) and id-based references work
under this rule. The exact storage-format snippets for both options are
in
[reference.md § Set-mode de-duplication via reading-order references](reference.md#set-mode-de-duplication-via-reading-order-references).

Forward references (parent → child) and prose-only pointers (the
deprecated "see also: the runbook section X" fallback used when no
update tool was available) are **never** emitted under the current
rule. If you find yourself about to emit one, the owner assignment was
wrong — re-run Step 5 with parent-priority enforced.

## Step 7 — Empty target sections

A target section that received no source blocks gets exactly one line:

```
_Empty — content missing._
```

Do not pad it. Do not retitle it. Do not hide it. Do not editorialise. Empty
sections are low-importance but must remain visible so the gap shows up
honestly in Step 9.

## Step 8 — Discrepancies section

Append one new top-level section at the end of the body, titled:

```
Discrepancies — content not matching the template
```

Group excess blocks by `topic`. Each block keeps its source-URL footnote.
Phrasing stays neutral: this section relocates content, it does not
comment on it.

**Set mode**: each excess block lives in **exactly one** draft's
Discrepancies section — the draft whose below-threshold score was highest
for that block (see Step 5). Other drafts do **not** repeat the excess
block, neither as a reference nor as a copy. This keeps the excess
character-count bookkeeping in Step 9 clean (no block is counted twice
across the set) and avoids three or four near-duplicate Discrepancies
sections across the drafts.

If a draft ends up with no excess at all, omit the Discrepancies section
on that draft entirely.

## Step 9 — Coverage footnote

Append a final footnote at the very end of the document, titled
`Mapping coverage`. It MUST be **concise** (no narrative, just the numbers
and the offending items) and MUST contain the metrics below.

**Missing% (section-count basis)**

```
missing% = empty_target_sections / total_target_sections * 100
```

A section is "empty" when neither an owned block nor a reference landed
there. Phrase it like the operator's own examples — for instance, "10% of
the information is missing (1 of 10 target sections empty)". Then list the
empty sections by name.

**Excess% (character-count basis, per topic and overall)**

For **single mode** and for each draft in **set mode** (where this draft
holds the excess block — only one Discrepancies destination per block, by
Step 8):

```
excess%(topic) = excess_chars(topic)
                 / (excess_chars(topic) + retained_chars(topic))
                 * 100

excess%(overall) = sum(excess_chars) / (sum(excess_chars) + sum(retained_chars)) * 100
```

In set mode, `retained_chars` counts **owned chars in this draft only**.
Referenced blocks do NOT contribute to `retained_chars` — their body
lives on the owner draft and is counted there. This keeps the per-draft
Excess% honest: a draft consisting mostly of `excerpt-include` macros
will have a small `retained_chars` denominator and the metric will
reflect that.

Phrase each topic line like the operator's own example — for instance,
"66% of the documentation on topic X does not make it into the final
document" (which is what you get when excess is twice the retained
volume). Then list every excess block with its source URL.

The two metrics are deliberately not symmetric: missing% is measured by
**section count** (structure coverage), excess% is measured by
**character count** (volume of source content that did not survive the
mapping). Both must appear.

**Reference%** (set mode only, optional)

```
reference% = referenced_sections / total_target_sections * 100
```

Share of this draft's sections that are reference-only (an
`excerpt-include` macro and nothing else). High Reference% means the
draft is closer to a curated index than a standalone document; useful
context for the reader. Print one line if Reference% > 0; skip the
metric entirely otherwise.

See [reference.md](reference.md) for a full worked example.

## Step 10 — Publish

**Title inference (new pages only).** Build the destination title from the
template title captured in Step 4 and the feature name captured in Step 1.4:

```
destination_title = "<feature name> - <stripped template title>"
```

Use the operator's original casing for the feature name (not the slug from
Step 11). Before substituting, strip from the template title, in this order:

1. Any **leading reading-order index** detected in Step 1.5 — the same
   regex match, including its trailing separator (`1. ` `2.1 ` `[3] `
   `01 - ` `A. ` `B1) `). The destination title must not carry the index;
   the order is recorded in the consolidated plan and the overview table,
   not in the title itself.
2. Any placeholder marker that wraps the title — `[…]`, `<…>`, `{{…}}`,
   leading/trailing literal "Template" or "Modèle".

Examples:

- template `1. Functional Documentation` + feature `Auth token refresh`
  → `Auth token refresh - Functional Documentation`
- template `2.1 Data flow` + feature `Auth token refresh`
  → `Auth token refresh - Data flow`
- template `Architecture Decision Record [Template]` + feature
  `Auth token refresh` → `Auth token refresh - Architecture Decision Record`
- template `{{Service runbook}}` + feature `Billing webhook` →
  `Billing webhook - Service runbook`

In set mode, drafts are created **in the confirmed reading order** (Step 1.5)
so the `#` recorded in the overview table is stable across the run.

Always show the inferred title back to the operator and let them confirm or
override before calling the publish tool.

For an **existing page** being updated, leave the title untouched unless the
operator explicitly asks to rename it.

- New page → `createConfluencePage` under the chosen parent with the
  confirmed title.
- Existing page → `updateConfluencePage`. This branch is only reachable
  when Step 2.2 confirmed the tool is available on the discovered MCP;
  if it was missing, the run already stopped in Step 2 and asked the
  operator to switch the destination to a new page.

Print the resulting page URL back to the operator before exiting.

## Step 11 — Tag the destination

The destination MUST be tagged with three labels, in this order:

1. `<feature-name-slug>` from Step 1.4 (e.g. `auth-token-refresh`).
2. `technical` or `functional` from Step 1.6 (per template in set mode).
3. `cursor` (constant — marks pages produced by this skill).

**Only do this when the destination supports labels/tags/metadata.** Decide
support, in this order:

1. The destination's MCP exposes a label-management tool (e.g.
   `addConfluenceLabel`, `addLabelsToConfluencePage`, `setPageLabels`,
   anything matching `*Label*`). Use it.
2. The publish tool itself accepts a `labels` argument. Pass them there.
3. Neither is available → **fallback**: prepend a small visible line to the
   page body so the labels are at least textually searchable, AND tell the
   operator labels could not be applied as native metadata. Example body
   prefix:

   ```html
   <p><em>Labels:</em> <code>auth-token-refresh</code> &middot;
   <code>technical</code> &middot; <code>cursor</code></p>
   ```

For Confluence specifically, the project's currently-discovered MCP tools
do **not** expose label management — verify with
`ls mcps/<server>/tools/ | findstr /I label` before assuming otherwise. If
nothing matches, take the fallback path and surface the limitation to the
operator. See [reference.md](reference.md#tagging-the-destination) for full
detection rules and per-destination guidance.

In set mode, the overview page from Step 12 carries `overview` as its
`doc kind` label instead of `technical` / `functional`.

## Step 12 — SET MODE ONLY: publish the timestamped overview/coverage summary

After all N per-template runs complete, generate **one additional concise,
timestamped summary** that indexes the N drafts and reports aggregate
metrics across the set.

### 12.0 — Ask where the summary lands

Before producing anything, ask the operator (use `AskQuestion` when
available):

```
"Where should the timestamped summary land?"
  - Local markdown file (default)
    → ask for an absolute path; default to
      <repo-root>/scratch/<feature-slug>-overview-<UTC timestamp>.md
  - Confluence page
    → publish under the same parent as the N drafts (Step 1.3)
  - Both
```

The summary is the **same content** in either destination, only the
rendering differs (markdown for local, storage format for Confluence).

### 12.1 — Title and timestamp

Capture a single UTC timestamp at the start of Step 12 and reuse it
everywhere (filename, page title, header).

```
timestamp_utc    = "<YYYY-MM-DD HH:MM UTC>"
overview_title   = "<feature name> - Documentation overview & coverage (<timestamp_utc>)"
```

### 12.2 — Required content (concise, in this order)

The summary MUST stay **concise** — terse tables, short bullets, no
narrative padding. Same outline whether it lands locally or in Confluence:

1. **Header** — feature name, timestamp, source description, run mode.
2. **The N drafts** — one-row-per-draft table, **rows in the confirmed
   reading order from Step 1.5** so `#` matches the order an engineer
   should read the set:
   `# · Rank index · Template · Doc kind · Title · Page ID · Status · URL ·
   Missing% · Excess%`. Optional one-line summary per draft (skip when
   the table already speaks for itself).
3. **Aggregate coverage (ownership-aware)** — `total_chars`, `owned_chars`,
   `not_covered_chars`, `aggregate_coverage_pct`, `not_covered_pct`. A
   block counts as covered iff some draft **owns** it; pure references do
   not add coverage. One short bullet list itemising the residual by
   source page.
4. **Reference map (replaces the redundancy section)** — for each draft,
   `owned_chars`, `reference_chars_into`, and a short
   "reads excerpts from …" summary. The aggregate `dedup_ratio_pct`
   number on one line. A pairwise reference-flow table (`D_a → D_b`,
   chars pulled, sections involved), sorted by chars pulled, only
   non-zero rows.
5. **Per-source-page perspective** — `page_coverage_pct(p)` + the spread
   of owners across drafts for that page (one row each).
6. **Per-template comparison** — `Template · Sections · Missing% ·
   Excess% · Reference% · Owns (chars) · Pulls in (chars)` (one phrase
   per cell where applicable).
7. **Reading the numbers** — 2–3 bullets max (high dedup_ratio = tightly
   coupled drafts; near-zero = independent; outsized residual; any draft
   whose pulled-in references dwarf its owned body).
8. **Caveats** — one bullet: numbers are directional (placements map,
   not re-fetch + diff). One bullet: excerpt-include macros only resolve
   once every owner page is published — single-draft previews mid-run
   will show "page not found" for some macros. Add the label-fallback
   note if it applies.
9. **Follow-ups** — 3–5 bullets max (apply native labels, promote drafts
   to `current` in publish order, review cross-rank ownership cases,
   investigate any draft with `dedup_ratio` > 50%).

Compute aggregate coverage and the reference-map metrics with the
formulas in
[reference.md § Documentation set mode](reference.md#documentation-set-mode);
they require keeping the ownership-aware `placements` map per source block
during Steps 5–9 of each per-template run.

### 12.3 — Render and persist

- **Local markdown** — write to the chosen path. If the file already
  exists, never overwrite — the timestamp suffix should already make
  collisions impossible; if one happens, append `-2`, `-3`, … and tell
  the operator. Print the absolute path back at the end.
- **Confluence page** — publish via `createConfluencePage` under the
  Step 1.3 parent and tag it with `<feature-slug>`, `overview`, `cursor`
  (Step 11 applies, including the fallback path).
- **Both** — produce the local file first (cheap, lets the operator
  preview), then publish the Confluence page from the same content.

Print, at the very end, the destination(s) of the summary AND the URLs of
all N drafts back to the operator.

## Additional resources

- [reference.md](reference.md) — Atlassian MCP tool index, Confluence storage
  format notes, AskQuestion templates, a fully worked metrics example, and
  the full Documentation set mode reference (formulas, overview-page
  template, worked example).
