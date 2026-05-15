# Documentation Mapper — Reference

Detailed material for the `documentation-mapper` skill. Read this when SKILL.md
points you here.

## Atlassian MCP tool index

Discover the server id with `ls mcps/*/tools/getConfluencePage.json` (the
parent directory of the match is the id). The identifier varies per
workspace — observed patterns include `user-atlassian` (user-scoped MCP)
and `project-<slug>-atlassian` (project-scoped MCP).

| Tool | Required? | Used in step | Purpose |
|------|-----------|--------------|---------|
| `getAccessibleAtlassianResources` | optional sanity | 2 | Sanity-check that the MCP can reach the operator's Atlassian site. |
| `getConfluenceSpaces` | optional | 1 | Help the operator identify a space when they only know the name. |
| `getPagesInConfluenceSpace` | optional | 3 | Bulk source enumeration when a whole space was requested. |
| `getConfluencePage` | required | 3, 4 | Fetch a single page (source OR template). Returns title, body, URL. |
| `getConfluencePageDescendants` | optional | 3 | Walk a parent page's subtree as additional source. |
| `searchConfluenceUsingCql` | optional | 3 | Pull sources by query (label, text, last-updated, etc.). |
| `getConfluencePageFooterComments` | optional | 3 | Pick up review notes that hint at the canonical wording for a section. |
| `getConfluencePageInlineComments` | optional | 3 | Same, scoped to a specific paragraph. |
| `createConfluencePage` | **required** | 10 | Publish a brand-new destination page. |
| `updateConfluencePage` | conditional (only for "update existing page" destinations) | 10 | Overwrite an existing destination page. |
| any `*Label*` tool | optional | 11 | Apply native labels; absence triggers the visible-body fallback. |

Always pass the **discovered** `serverIdentifier` to `CallMcpTool`; never
hardcode a server id in the skill output, because it is workspace-specific.

## Destination MCP capability check

SKILL.md Step 2.2 probes the discovered MCP for the tools the publish-side
steps (10, 11) need. The probe is shell-driven so it works regardless of
the MCP's tool naming conventions.

### Probes to run

```
# REQUIRED — stop if missing
ls mcps/<server>/tools/createConfluencePage.json

# CONDITIONAL — only matters when Step 1.3 picked "update existing page"
ls mcps/<server>/tools/updateConfluencePage.json

# OPTIONAL — informs Step 11's label strategy
ls mcps/<server>/tools/ | findstr /I label      # POSIX: grep -i label
```

### Decision matrix

| `createConfluencePage` | `updateConfluencePage` | Operator's Step 1.3 choice | Action                                                                                  |
|------------------------|------------------------|----------------------------|-----------------------------------------------------------------------------------------|
| missing                | (any)                  | (any)                      | **Stop.** Tell the operator the project has no publish path. No silent fallback.        |
| present                | missing                | "new page(s) under <parent>" | Proceed. Update branch is unreachable but unused.                                       |
| present                | missing                | "update existing page <id>"  | **Stop.** Two viable next moves: switch destination to "new page(s)", or cancel.        |
| present                | present                | (any)                      | Proceed.                                                                                |

For the label tool, the absence is **never** a stop — it routes Step 11 to
the visible-body fallback (see `## Tagging the destination` below). The
skill must surface the absence to the operator at probe time, not hide it
until publication.

### Capability summary echoed to the operator

After the probe, print one short line so the operator can sanity-check
before any source is fetched:

```
Capabilities (server <discovered id>):
  createConfluencePage   [ok]
  updateConfluencePage   [missing — update branch disabled]
  label management       [missing — Step 11 will use the body fallback]
```

This summary is intentionally minimal — no narrative, no recommendations
the operator has not asked for.

### Worked example — `user-atlassian` MCP

For an Atlassian MCP exposing the user-scoped tool surface seen in
practice (`user-atlassian` in this workspace, May 2026):

- `createConfluencePage` → present.
- `updateConfluencePage` → **missing**.
- `*Label*` → no matches.

→ Skill proceeds only when Step 1.3 picked "new page(s) under <parent>".
Step 11 takes the visible-body fallback. The operator is told both facts
at probe time.

## AskQuestion templates

Step 0 (mode selection) is one block, asked first:

```
0. "Run mode?"
   - single  — one template → one document (default)
   - set     — N templates → N documents + an overview/coverage page
               cross-referencing them
```

Default to `single` if the operator's request mentions a specific doc kind;
default to `set` if they ask to "document the feature" / "produce the doc
set" / mention several angles. Always confirm the inferred default.

Step 1 is then driven by these blocks (5 in single mode, 6 in set mode):

```
1. "What is the source of the documentation to map?"
   - Confluence (default)
   - Jira
   - Local Markdown / files
   - Web URLs
   - Mixed (operator will list)

2. "How should I locate the source(s)?"  (Confluence-specific follow-up)
   - One or more page IDs
   - A parent page ID (I will pull descendants)
   - A CQL query
   - A whole space (warn the operator about volume)

3. "Where should the new document(s) land?"
   - New page(s) under <parent page ID>  (title will be inferred in Step 10
     from "<feature name> - <template title>" and confirmed before publish;
     in set mode all N drafts AND the overview page land under the same
     parent)
   - Update existing page <page ID>   (single mode only; title left untouched)

4. "What is the feature name?" (free text; propose a default if you can
   infer it from the chat — e.g. operator said "document the auth refresh
   flow" → propose "auth refresh flow")

5a. "Where do the templates live?" (one blocking prompt, asked before
    scanning anything)
   - A folder or parent page
       → operator gives one Confluence folder or page ID; in set mode the
         skill expands its direct children (via
         `getConfluencePageDescendants` for a page, or
         `searchConfluenceUsingCql parent = <id>` for a folder) and
         presents the children for multi-select. In single mode, if
         exactly one child is found, use it; otherwise the operator
         picks one.
   - A list of distinct page/template IDs
       → operator pastes one or more page/template IDs explicitly. No
         expansion is performed. The list order is recorded as a
         **reading-order fallback** for Step 1.5 (used only if
         index-detection cannot infer the hierarchy).
   - Auto-scan the destination space  (set mode only)
       → CQL: `space = <KEY> AND label = "template"`, plus a title-LIKE
         fallback (`*template*`, `<feature name>*`). Skipped in single
         mode — too risky to auto-pick.

5b. "Confirm the template list." (set mode only)
    Show the resolved candidates as
    "Title · Page ID · top-level sections (preview)" and let the
    operator multi-select / drop / re-order before continuing. In
    single mode, show the single chosen template and let the operator
    confirm or pick a different one.

6. "Doc kind for each template?" (asked once in single mode, once per
   template in set mode)
   - technical
   - functional
   Default per template title:
     - contains "functional"                     → functional
     - contains "runbook"|"operations"|"design"|
       "architecture"|"deployment"|"testing"     → technical
   Always offer the default and let the operator override.

7. (set mode only) Reading-order prompt — one of two flavours,
   depending on whether index-detection succeeded
   (see "Detecting the reading order from template titles" below):

   - **Inferrable** — every template has a unique, non-ambiguous index.
     Print the consolidated plan sorted by rank and ask:
       - Confirm           — proceed with the detected order
       - Re-rank manually  — operator restates the order
       - Cancel

   - **Not inferrable** — no leading index, mixed indexed/unindexed
     templates, or duplicate indices. Block with an explicit prompt
     that states the failure reason and requires the operator to
     specify the order:
       "I could not infer the reading order from the template titles.
        Reason: <no leading index | duplicate indices: <list> |
        mixed indexed/unindexed: <list>>.
        Please specify the reading order explicitly (rank 1 → rank N)."
     Loop until every template has a unique rank in 1..N. If the
     operator chose "List of distinct page/template IDs" in 5a,
     propose that input order as the default. Cancel is always
     available.
```

Set mode adds one final block at the start of Step 12:

```
7. "Where should the timestamped overview/coverage summary land?"
   - Local markdown file (default)
       → ask for an absolute path; default to
         <repo-root>/scratch/<feature-slug>-overview-<UTC timestamp>.md
   - Confluence page (under the same parent as the N drafts)
   - Both
```

Always echo the operator's choices back before running anything destructive.
In set mode, confirm the consolidated plan as a small table **sorted by the
detected reading order**, with the `Rank index` column visible so the
operator can spot bad detections (false positives like `QA Procedures`):

```
# · Rank index · Template title                    · Page ID    · Doc kind  · Inferred title
1 · 1          · 1. Functional Documentation       · 66947548021 · functional · Auth refresh - Functional Documentation
2 · 2          · 2. Solution / Design              · 66948498160 · technical  · Auth refresh - Solution / Design
3 · 3          · 3. Runbook / Operations           · 66948596304 · technical  · Auth refresh - Runbook / Operations
4 · ?          · Glossary                          · 66948596400 · technical  · Auth refresh - Glossary
```

Print any **mixed-schemes** or **duplicate-index** warnings on the lines
immediately above the table — see the next subsection for what triggers
them.

## Detecting the reading order from template titles

Set mode (Step 1.5) sorts the drafts by a **leading alpha-numerical index**
parsed from each template title. The same index is also stripped from the
title in Step 10 (the destination title carries no rank prefix). Validated
on a representative test corpus in `scratch/test-ranking.js`.

### Regex

```
^[\s\[(]*((?:\d+[A-Za-z]?|[A-Za-z]{1,2}\d*)(?:[.-](?:\d+[A-Za-z]?|[A-Za-z]{1,2}\d*))*)[\s.):\]}-]+(?=\S)
```

What it captures:

- Pure digits: `1`, `10`, `01`
- Hierarchical: `1.1`, `1.2.3`, `2-1`
- Letter codes: `A`, `AA`, `B1`, `1A`
- Bracketed: `[3] …`, `(1) …`
- Padded: `01 - …`
- Mixed separators: `B1)`, `B2:`, `2.1 `, `[3] `

Anchored to the start; requires a separator (`.` `)` `:` `]` `}` `-`
whitespace) AND non-whitespace after it, so titles like `Service runbook`
or `Architecture` cannot match.

### Sort key & natural-sort algorithm

Split the captured index on `.` or `-`. For each segment:

- pure digits → `{ cls: 0, num: parseInt(seg, 10) }` (numeric)
- otherwise   → `{ cls: 1, str: seg.toUpperCase() }`  (alpha)

Compare two keys segment-by-segment:

- The shorter key sorts first when one is a prefix of the other (so `1`
  before `1.1`).
- Same depth, different class → **numeric sorts before alpha**.
- Same depth, same class → compare numeric value (numeric) or
  upper-cased string (alpha).

Examples (ascending):

```
1 · 1.1 · 1.2 · 1.10 · 2 · 2.1 · 10 · A · A1 · AA · B
```

Unindexed templates get a null key. Their treatment depends on the
overall set:

- If **all** templates are unindexed, detection has no signal at all
  and triggers the "No detectable index on any template" hard failure
  in the next section.
- If **some** templates are indexed and others are not, detection
  triggers the "Mixed indexed/unindexed templates" hard failure — the
  null-keyed ones are not silently appended.

Only when every template carries a non-null key (and there are no
duplicates) does the natural sort produce the final reading order.

### Detection outcomes — soft warnings vs. hard failures

Hierarchy detection produces one of three outcomes; each drives a
different operator prompt in Step 1.7 above.

#### Soft warning — hierarchy is still inferrable

- **Mixed schemes** — at least one indexed template has a numeric
  top-level segment and another has an alpha top-level segment. The
  skill sorts numeric first and alpha second, but the operator should
  confirm this is what they want. Surface a one-line warning above the
  consolidated plan table; the **Confirm / Re-rank manually / Cancel**
  prompt is still offered (no hard block).

#### Hard failure — hierarchy is NOT inferrable, operator must specify it

Any of the following triggers the "Not inferrable" branch of Step 1.7:

1. **No detectable index on any template** — the leading-index regex
   matches zero of the picked templates. There is no signal to sort on.
2. **Mixed indexed/unindexed templates** — some templates carry a
   leading index, others don't. The skill cannot place the unindexed
   ones reliably; falling back to "append unindexed at the end with
   rank `?`" hides the ambiguity.
3. **Duplicate index** — two or more indexed templates resolve to the
   same sort key (e.g. `(1)` and `01`, or `A.` and `A)`). There is no
   unique ordering.

For each hard failure, surface BOTH the failure type and the offending
templates (raw titles + detected indices + row numbers) in the
operator prompt. Do not propose a default ordering for the operator to
"Confirm" — require them to specify the order explicitly (rank 1 →
rank N). If the operator picked option 5a "List of distinct
page/template IDs" in Step 1, propose the input order as the default
suggestion to copy, but still require explicit confirmation.

Neither outcome is silently swallowed: soft warnings are visible above
the consolidated plan, hard failures block until the operator answers.

### Known false-positive: 2-letter abbreviations

Because alpha indices can be 1–2 letters, **any title that starts with a
1–2-letter token followed by a separator gets captured as an index**.
Examples seen in the wild:

- `QA Procedures` → captured as `QA` (false positive)
- `Ops Runbook`   → captured as `Op` … wait — `Ops` is 3 letters and won't
  match. **Only 1–2-letter prefixes are at risk.** The most common
  offenders are `QA`, `UX`, `AI`, `ML`, `IT`, `HR`, `BI`.

Mitigation:

- The consolidated plan table shows `Rank index` per row — the operator
  sees the false positive immediately.
- The Step 7 AskQuestion offers `Re-rank manually` so the operator can
  override.
- 3+ letter prefixes (`API`, `SDK`, `CRM`) are correctly rejected.

### Title inference rule (Step 10)

```
destination_title = "<feature name> - <stripped template title>"
```

Apply these strips to the template title in order:

1. **Leading reading-order index** detected in Step 1.5 — the entire regex
   match including its trailing separator. So `1. Functional Documentation`
   → `Functional Documentation`, `2.1 Data flow` → `Data flow`,
   `[3] ADR — Token refresh` → `ADR — Token refresh`, `B1) Runbook`
   → `Runbook`. The reading order lives in the consolidated plan and the
   overview drafts table — never in the destination title.
2. **Placeholder wrappers**: `[…]`, `<…>`, `{{…}}`, leading/trailing
   `Template` or `Modèle`.

Other rules:

- Use the operator's original casing for `<feature name>`, NOT the slug.
- Always confirm the inferred title with the operator before publishing.
- Existing-page updates: title is left unchanged unless the operator
  explicitly asks to rename.

## Confluence storage-format notes

- Pages are stored in **Confluence Storage Format** (XHTML-ish).
  Headings are `<h1>` … `<h6>`. Don't drop heading levels when copying source
  blocks into the target; renumber so the target's hierarchy stays clean.
- Footnotes: prefer `<ac:structured-macro ac:name="footnote">` if the site has
  the macro installed. Otherwise emit numeric superscripts (`<sup>[1]</sup>`)
  and a `<h2>References</h2>` block listing the URLs.
- The `Empty — content missing.` line should be a single italic paragraph
  (`<p><em>Empty — content missing.</em></p>`). No callout, no panel, no
  warning icon — keep it visually quiet, exactly as SKILL.md requires.
- Both `createConfluencePage` and `updateConfluencePage` (when present —
  see § Destination MCP capability check) accept storage-format bodies.
  Validate that every `<ac:*>` macro you emit is opened and closed.

## Block splitting heuristic

A "block" is the smallest unit that deserves its own footnote. Practical rule:

- A heading + everything until the next heading of equal or higher level → one
  block.
- A standalone table or fenced code sample inside that range, longer than
  ~10 lines → split into its own block (so it gets its own footnote).
- Bullet lists stay attached to their introducing paragraph unless they exceed
  ~15 items.

`char_count` is computed on the **rendered text** of the block (strip
storage-format tags first), because the excess metric is a measure of
information volume, not markup volume.

## Worked metrics example

Template has **10** sections. After mapping:

- **9** sections received at least one source block.
- **1** section received nothing → marked `_Empty — content missing._`.
- Across all sources, **3 000** characters of content matched the template
  (retained) and **6 000** characters did not (excess), all on a single topic
  `Topic X`.

Coverage footnote should read approximately:

```
Mapping coverage

Missing
  10% of the information is missing (1 of 10 target sections empty).
  Empty sections:
    - 4. Security considerations

Excess
  Topic X — 66% of the documentation on topic X does not make it into the
  final document (6 000 excess chars vs. 3 000 retained chars).
  Overall — 66% of source volume did not survive the mapping
  (6 000 / 9 000 chars).

  Excess blocks:
    - "Legacy provisioning script" — https://example.atlassian.net/wiki/...
    - "Vendor integration notes"   — https://example.atlassian.net/wiki/...
```

The arithmetic checks out: `6 000 / (6 000 + 3 000) = 0.6666… ≈ 66 %`, which
matches the operator's stated example ("twice as much unidentifiable data as
the final volume → 66 %").

## Tagging the destination

Step 11 of SKILL.md mandates three labels on the destination, in this order:

1. `<feature-name-slug>`
2. `technical` or `functional`
3. `cursor`

### Slugifying the feature name

```
slug = feature_name.lower()
slug = re.sub(r'[^a-z0-9]+', '-', slug).strip('-')
slug = slug[:50]
```

Examples:
- `"Auth Token Refresh"`        → `auth-token-refresh`
- `"OAuth2 / PKCE flow"`        → `oauth2-pkce-flow`
- `"Réécriture du moteur 2.0"`  → `r-criture-du-moteur-2-0` *(strip diacritics
  first if you want a cleaner slug — optional, but be consistent)*

### Detecting label support

Run these checks in order; stop at the first that succeeds.

1. **Dedicated tool.** Look in the discovered MCP for a tool whose name
   matches `*Label*` (case-insensitive):

   ```
   ls mcps/<server>/tools/ | findstr /I label
   ```

   Likely names: `addConfluenceLabel`, `addLabelsToConfluencePage`,
   `setConfluencePageLabels`. Read its descriptor before calling, then call
   it once per label (or once with all three, depending on the schema).

2. **Inline argument on the publish tool.** Re-read whichever of
   `createConfluencePage.json` / `updateConfluencePage.json` are present
   (per Step 2.2's capability summary) and check for a `labels`, `tags`,
   or `metadata` property. If present, pass the three labels there and
   skip step 11's separate call.

3. **Fallback (current state for the project's Atlassian MCP).** Neither
   path is available. Then:

   - Prepend a `<p><em>Labels:</em> ...</p>` line to the body so the labels
     are at least full-text-searchable in Confluence.
   - In the operator-facing summary printed at the end, add one line:
     `Labels could not be applied as native Confluence metadata — emitted
     as a visible body line instead.`
   - Do NOT silently skip. The operator must know.

### Per-destination guidance

| Destination | How labels work |
|-------------|-----------------|
| Confluence (this project's MCP) | No label tool exposed → fallback path. |
| Confluence (other projects' MCPs) | Re-run the detection above; tool names are not standardised. |
| Jira issue | `editJiraIssue` accepts a `labels` field — pass the three labels there. |
| Local Markdown file | Emit YAML frontmatter `labels: [feature-slug, technical, cursor]`. |
| Generic web destination | No metadata channel → fallback path (visible labels line). |

## Edge cases

- **No template content examples.** Match on heading similarity alone, but
  raise the confidence threshold so weakly-matching blocks correctly fall into
  Discrepancies rather than being force-fitted.
- **Source heading collides with target heading.** The collision is evidence
  for a strong match — boost the score, don't ignore it.
- **Same source block fits two target sections.** Place it in the deepest
  match; add a one-line cross-reference (`See also: …`) in the other section.
  Never duplicate the body — duplication breaks the character-count
  bookkeeping.
- **Operator picks a Confluence template (not a page).** Templates are fetched
  the same way as pages once you have the template id; their bodies typically
  contain richer placeholder text, which is good signal for Step 5.
- **Mixed sources where some have no URL.** Use `file://` or `cql://<query>`
  pseudo-URLs in the footnote so every block still has a citation.

## Non-Confluence sources (later iterations)

The skill is designed Confluence-first but the mapping core is source-agnostic.
For Markdown / PDF / web sources, replace Step 3 with the appropriate fetcher
and keep the same block schema (`source_url`, `source_heading`, `char_count`,
`raw`). Steps 4–10 remain unchanged.

## Documentation set mode

Set mode is the multi-template specialisation chosen in Step 0. It runs the
single-doc workflow N times against the same source set, then publishes one
extra page (Step 12 of SKILL.md) that indexes the N drafts and reports
aggregate metrics across the whole set.

### Set-mode de-duplication via reading-order references

Each source block has at most **one owner** across the entire set; every
other draft that wants the same block emits a cross-reference macro
pointing at the owner. The body lives in exactly one place.

#### Two coupled axes — parent-priority

| Axis              | Definition                                                                                                                                  | Why it matters                                                       |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| **Hierarchy**     | Reading-order from the template-title index (Step 1.5). Rank-1 is the "parent" of rank-2 of … .                                              | Defines the order the operator/reader walks the set, AND drives ownership (see below). |
| **Ownership**     | Per-block, the **earliest-ranked** template (lowest rank index) whose best section scores ≥ threshold. Tie-breaker within a draft: section depth. | Defines where the body of a given block physically lives. The parent draft is the source of truth whenever it has a matching section. |

The two axes are coupled by design (operator decision 4 — "the parent is
the source of truth, even when a child is semantically a better fit").
As a consequence:

- A parent draft **never references a child draft.** The reference
  graph is strictly backward (later rank → earlier rank) and acyclic.
- Owners are always published before their referencers (Step 10
  publishes in reading-order rank), so backward references always
  resolve at child-creation time.
- This is what allows the rendering options in
  § Rendering the reference in Confluence storage format to be
  interchangeable — plain backward smart links work as well as
  `excerpt-include` macros, because no draft ever points at a not-yet-
  published owner.

A consequence the operator should expect: the rank-1 draft accumulates
**most** of the body content when its template covers a broad range of
section types. Later-ranked drafts become reading-order-specific views
that re-narrate the parent through references. High `reference%` on
child drafts is expected and healthy under this rule, NOT a smell.

#### Algorithm (parent-priority)

For each source block `B` (block-major, not draft-major):

```
# All drafts, in reading-order rank ascending (rank-1 first).
drafts_in_rank_order = sorted(drafts, key=lambda d: d.rank)

owner_draft   = None
owner_section = None

# Walk parents first; the first draft that has a fitting section wins.
for d in drafts_in_rank_order:
    d_best_section, d_best_score = argmax(score(B, s) for s in d.sections)
    if d_best_score >= threshold:
        owner_draft   = d
        owner_section = d_best_section          # ties within d: deeper wins
        break

if owner_draft is not None:
    owner_excerpt = slugify(owner_section)      # named-excerpt key
    placements[B].owner_draft   = owner_draft
    placements[B].owner_section = owner_section
    placements[B].owner_excerpt = owner_excerpt
    placements[B].drafts.append({owner_draft, owner_section, "owned"})

    # Children only — every draft ranked AFTER the owner whose own best
    # section still scores >= threshold gets a backward reference.
    # Earlier-ranked drafts (impossible by construction here) and the
    # owner itself are skipped, so no parent ever references a child.
    for d in drafts_in_rank_order:
        if d.rank <= owner_draft.rank:
            continue
        d_best_section, d_best_score = argmax(score(B, s) for s in d.sections)
        if d_best_score >= threshold:
            placements[B].drafts.append({d, d_best_section, "referenced"})

else:
    # Block fits no draft well enough — goes to ONE Discrepancies section,
    # owned by the draft whose below-threshold score was highest, so
    # excess content is never duplicated either. We pick the best
    # below-threshold (any rank) because there is no owner to anchor on.
    best_below = argmax((d, score) for d in drafts for score in below_threshold)
    placements[B].owner_draft = None
    placements[B].drafts.append({best_below.draft, None, "excess"})
```

Why walk ranks in order instead of taking the global argmax?

- It enforces **parent-priority**: a child draft's higher score never
  beats a parent draft that is "good enough" (≥ threshold). The parent
  becomes the source of truth.
- It makes the **reference graph strictly backward**: by the time the
  inner loop runs, only `d.rank > owner_draft.rank` drafts are still
  candidates. Cycles are eliminated by construction, not by post-hoc
  detection.
- It allows backward references to be rendered as **plain smart links**
  (a cheaper, more portable alternative to `excerpt-include` macros)
  because the owner is always published before any draft that
  references it — Confluence does not have to resolve a not-yet-existing
  page.

Implementation notes:

- Scoring is the same as single mode (heading-similarity + example-content
  similarity), just evaluated globally instead of per-draft. The threshold
  is the same.
- Within a single draft, ties between sections at the same score are
  broken by section depth (deeper wins).
- A child draft that has no section ≥ threshold for a given block emits
  **nothing** for that block — neither a reference nor a placeholder.
  The block exists only on the owner page.
- Step 4 must complete for **all** templates in the set **before** Step 5
  runs — the per-draft argmax still needs every section visible.

#### Rendering the reference in Confluence storage format

Owners wrap the section body in a **named excerpt** so other pages can
include it:

```html
<h2>{owner_section}</h2>
<ac:structured-macro ac:name="excerpt">
  <ac:parameter ac:name="name">{owner_excerpt}</ac:parameter>
  <ac:rich-text-body>
    {body of block B, with source-URL footnote as in single mode}
  </ac:rich-text-body>
</ac:structured-macro>
```

References emit one short line in the matching section of the child
draft:

```html
<h2>{child_section}</h2>
<p><em>This section is owned by <a><ac:link>
  <ri:page ri:content-title="{owner destination title}"/>
  <ac:plain-text-link-body><![CDATA[{owner destination title}]]></ac:plain-text-link-body>
</ac:link></a> §{owner_section}. Excerpted below for context:</em></p>
<ac:structured-macro ac:name="excerpt-include">
  <ac:parameter ac:name="">{SPACE_KEY}:{owner destination title}</ac:parameter>
  <ac:parameter ac:name="name">{owner_excerpt}</ac:parameter>
  <ac:parameter ac:name="nopanel">true</ac:parameter>
</ac:structured-macro>
```

##### Option B — Plain backward smart link (interchangeable with A)

When named excerpts aren't supported, or when the operator prefers a
shorter visual footprint, the child draft emits one short italic
sentence with an inline smart link. The owner does NOT need to wrap the
section in any macro under this option.

```html
<h2>{child_section}</h2>
<p><em>Owned by the parent page. Open the canonical section directly:
  <a href="{owner_page_url}#{owner_section_anchor}"
     data-card-appearance="inline">{owner destination title} §{owner_section}</a>.</em></p>
```

In legacy storage format, swap `<a href="…" data-card-appearance="inline">`
for `<ac:link ac:anchor="{owner_section_anchor}">` wrapping the same
`<ri:page>` / `<ac:plain-text-link-body>` pair used in Option A's intro
line, without the trailing `excerpt-include` macro.

##### Deep-link rule (applies to both options)

Every backward reference MUST resolve to the **owner section anchor**,
not the owner page URL alone. The reader's click should land on the
canonical heading, not at the top of the parent page where they have
to scroll.

- **Option A (`excerpt-include`)** — the macro names the owner's
  excerpt explicitly via `<ac:parameter ac:name="name">{owner_excerpt}
  </ac:parameter>`. That parameter is the deep link by construction;
  no URL fragment is required.
- **Option B (plain smart link)** — the `href` MUST include
  `#{owner_section_anchor}`. A bare `href="{owner_page_url}"` (no
  fragment) is a **forbidden shortcut**: it is a sign the renderer
  skipped the anchor lookup and the reference must be regenerated.

###### Anchor algorithm

Given an owner section heading `H` as it is rendered on the published
owner page (after stripping markdown formatting like `_…_` or `**…**`
that the editor would not display in the final heading), the anchor
`A = anchor_of(H)` is built by:

```
A = H
A = strip from A every char in: & / ? ( ) [ ] { } < > " ' ! , ; : | * ` em-dash
A = collapse every whitespace run in A to a single "-"
A = trim leading and trailing "-"
# keep: letters, digits, "." , "_" , the dashes inserted above
# case is preserved (do NOT lowercase)
```

Examples:

| Owner heading text                              | `anchor_of(H)`                                  |
|-------------------------------------------------|-------------------------------------------------|
| `1. INTRODUCTION`                               | `1.-INTRODUCTION`                               |
| `7.2 Building a Linked Audience step-by-step`   | `7.2-Building-a-Linked-Audience-step-by-step`   |
| `6.6 Key rotation strategy (EMEA)`              | `6.6-Key-rotation-strategy-EMEA`                |
| `5.1 What is the Data Graph?`                   | `5.1-What-is-the-Data-Graph`                    |
| `10. OPERATIONAL NOTES / HANDOVER`              | `10.-OPERATIONAL-NOTES-HANDOVER`                |

###### Anchor-vs-heading hygiene (clean-heading rule)

Anchors get long and brittle if the owner heading carries template
noise. Render the owner heading with the **clean human heading text
only**:

- Drop `(from _SourceName_)` attribution suffixes, raw markdown
  wrappers, decorative emojis, trailing `[draft]` markers.
- Move all source attribution to a `_Source:_` line at the end of the
  owned section (or to a section-end footnote when there are several
  sources). The `_Source:_` line is what carries the `<a>`/`<ac:link>`
  back to the original source URL; the heading itself stays clean.
- This rule applies in single mode too — it costs nothing and makes
  the owner page nicer to read. It is **mandatory** in set mode
  because every child draft's reference URL embeds the heading's
  anchor.

###### Verifying anchors after the first publish

Different Confluence Cloud sites occasionally diverge on edge cases
(case folding, dot handling, em-dash treatment). After the first child
draft is published, click one of its `§N.M` references. If it scrolls
to the wrong section or to the page top:

1. Open the owner page in the browser, right-click the target heading,
   `Copy link` — Confluence pastes the exact URL with the site's
   actual anchor format.
2. Diff the pasted anchor against `anchor_of(H)` from the algorithm
   above. Adjust the algorithm (e.g. lowercase, dot-handling) to match
   the site.
3. Regenerate every reference using the corrected algorithm and
   re-publish the child drafts. Owners do not need to be re-published;
   only the references change.

Step 12's overview/coverage page documents the convention actually
used in its Caveats bullet, so the operator can verify against the
site once at run end rather than per-link.

Why both options are safe under parent-priority — and why
**title-based** and **id-based** references are interchangeable:

- Step 10 publishes drafts in reading-order rank, owner first. Every
  reference is backward, so by the time a child draft is created its
  owner already exists — Confluence resolves the link / macro at the
  moment of publish, not at view time later. No
  "page not found" caveat is needed.
- This is what eliminates the deprecated **prose-pointer fallback**.
  Earlier iterations of the skill emitted non-clickable
  "see also: §<section> on the runbook page" text when an owner page
  was not yet published (typically a parent → child cycle on a server
  without an update tool). Under parent-priority no draft ever needs
  to reference a not-yet-published owner, so prose pointers are dead
  code: **they are forbidden in the current rule**.
- Title-based references (`SPACE:<destination title>`) survive page-id
  changes (e.g. promotion from draft to current); id-based references
  survive title renames. Either is acceptable. Pick one and be
  consistent within a run.

#### Picking between Option A and Option B

Option A (`excerpt-include`) renders the owner body inline in the child
page, which is the richer reading experience but requires Confluence
Cloud's **named excerpts** feature (multiple `excerpt` macros per page,
each tagged with `<ac:parameter ac:name="name">…</ac:parameter>`).
Option B (plain backward smart link) works on every Confluence (Cloud
or legacy Server) and renders as an inline-card preview in the modern
editor.

Decide at Step 2.2:

- Probe for named-excerpt support with a one-off `getConfluencePage`
  call against a known excerpt-bearing page in the destination space,
  OR fall back to Option B unconditionally when in doubt. Both options
  are first-class under parent-priority.
- Be consistent within a run — don't mix the two options across the
  same set of drafts. Pick one rendering and use it for every
  reference.

For Option A, the `ac:anchor` (if you also include one for
accessibility on the `<ac:link>` intro line) follows the same
`anchor_of(H)` algorithm documented under the Deep-link rule above.
Note that the historical formulation ("lowercased, spaces → `-`,
non-alphanumerics stripped") understated Confluence Cloud's
case-preserving behaviour on the modern editor and stripped the dots
that Cloud actually keeps; the algorithm under the Deep-link rule
section above is the corrected one and is what every reference in
this skill must use.

### Per-block tracking schema

Set mode is **de-duplication-by-reference**: each source block has at most
**one owner** across the whole set, and every other draft that wants the
same block emits a cross-reference macro to the owner instead of copying
the body. See § Set-mode de-duplication via reading-order references
below for the algorithm; this section just defines the bookkeeping the
algorithm writes to.

A shared `placements` map is updated during Step 5 and keyed by
source-block id:

```
placements: {
  <block_id>: {
    "char_count":     <int>,                  # from Step 3
    "source_url":     "...",
    "owner_draft":    <0..N-1> | null,        # best-fit draft when score
                                              # >= threshold somewhere;
                                              # null when block is excess
                                              # everywhere
    "owner_section":  "<target heading path>",
    "owner_excerpt":  "<excerpt-name slug>",  # named-excerpt key used by
                                              # the include macro
    "drafts": [                               # one entry per draft that
                                              # touched this block
      { "draft_idx": <0..N-1>,
        "section":   "<target heading>" | null,
        "kind":      "owned" | "referenced" | "excess" }
    ]
  }
}
```

Invariants:

- For any block, **exactly zero or one** `drafts` entry has
  `kind: "owned"`. Zero ⇔ no draft scored ≥ threshold for this block
  ⇒ the block is excess somewhere.
- `kind: "referenced"` entries appear **only in drafts ranked strictly
  after the owner** (`d.rank > owner_draft.rank`). The
  parent-priority algorithm in § Algorithm guarantees this by
  construction — earlier-ranked drafts either own the block or do not
  touch it. A `referenced` entry on a draft with `rank <= owner.rank`
  is a bug; surface it in the overview's Reading-the-numbers section.
- The reference graph induced by `(owner_draft, referenced_drafts)`
  edges is **strictly backward** (later rank → earlier rank) and
  therefore **acyclic**. No cycle detection or breaking is needed
  downstream.
- `kind: "excess"` entries appear at most once per block, in the draft
  whose best-fit score was highest among the below-threshold scores
  (so excess blocks land in a single Discrepancies section, never
  duplicated across drafts).

Derived helpers:

- `owner_of(block)` ⇒ the unique `draft_idx` with `kind == "owned"`, or
  `null` if the block is excess.
- `references_to(owner_draft, block)` ⇒ list of `draft_idx` with
  `kind == "referenced"` for that block (where the macro lives).
- `is_covered_by_set(block)` ⇔ `owner_of(block) is not None`. A block
  is "covered by the set" iff some draft owns it; pure references do
  not contribute new coverage.

### Aggregate coverage formula (ownership-aware)

A block is "covered by the set" iff some draft owns it. References do not
add new coverage — they are pointers, not body content.

```
total_chars            = sum(block.char_count for block in source_set)
owned_chars            = sum(block.char_count for block in source_set
                             if owner_of(block) is not None)
not_covered_chars      = total_chars - owned_chars     # = excess chars

aggregate_coverage_pct = owned_chars / total_chars * 100
not_covered_pct        = 100 - aggregate_coverage_pct  # = excess share
```

Itemise `not_covered_chars` by source page so the operator sees what the
residual is (typically navigation scaffolding, decorative content, open
TODOs, comment metadata). This is what lands in §3 (Aggregate coverage) of
the overview page.

### Reference-map formulas (replaces the redundancy section)

Because of the single-owner rule, **classical character-count redundancy
is ≈ 0 by construction**. The interesting set-level metric is no longer
"how much was duplicated" but "how interconnected are the drafts" — i.e.
the reference map.

```
references_count          = sum(len(references_to(owner_of(b), b))
                                for b in source_set
                                if owner_of(b) is not None)

referenced_chars(owner_d) = sum(block.char_count for block in source_set
                                if owner_of(block) == owner_d
                                and len(references_to(owner_d, block)) >= 1)

owned_chars(d)            = sum(block.char_count for block in source_set
                                if owner_of(block) == d)

reference_chars_into(d)   = sum(block.char_count for block in source_set
                                if d in references_to(owner_of(block), block))

draft_footprint(d)        = owned_chars(d) + reference_chars_into(d)
                            # the "size" of draft d as the reader sees it,
                            # owned body + pulled-in excerpts
```

Pairwise reference map (which draft pulls how much from which other draft):

```
references_chars(owner_d, child_d) =
    sum(block.char_count for block in source_set
        if owner_of(block) == owner_d
        and child_d in references_to(owner_d, block))
```

A non-zero `references_chars(a, b)` means: draft `b` reads excerpts from
draft `a`. The map is **not symmetric** — `references_chars(b, a)` is a
separate quantity (and can also be non-zero; cross-references between two
drafts are normal).

Optional aggregate metric — **dedup ratio**, useful for operator
follow-ups:

```
naïve_footprint           = sum(owned_chars(owner_of(b))
                                + sum(b.char_count for r in references_to(...)))
                            # what the set would weigh if every reference
                            # had copied its block instead of linking
duplicated_avoided_chars  = naïve_footprint - sum(draft_footprint(d) for d in drafts)
                            # in our model this equals
                            # sum over referenced blocks of
                            # (n_refs - 0) * block.char_count
                            # because the body is stored once on the owner
dedup_ratio_pct           = duplicated_avoided_chars / max(naïve_footprint, 1) * 100
```

The dedup ratio expresses "how much body content we did NOT duplicate
because of the reference model". 0% = drafts are disjoint (every block
fits exactly one draft, no cross-references possible). 50%+ = the set
is heavily cross-referenced; consider promoting strongly-coupled drafts
to siblings or a single combined doc.

### Per-source-page coverage

For each source page `p`, compute:

```
page_coverage_pct(p)        = sum(block.char_count for block in p
                                  if owner_of(block) is not None)
                              / sum(block.char_count for block in p)
                              * 100

most_referenced_owner(p)    = argmax_{d} sum(block.char_count for block in p
                                             if owner_of(block) == d
                                             and len(references_to(d, block)) >= 1)
```

This populates the "Per-source-page perspective" table on the overview
page.

### Overview / coverage summary template

The same outline is produced in two flavours depending on Step 12.0:

- **Markdown** — written to a local file. Use the block below verbatim.
- **Confluence storage format** — the publish tool converts markdown
  cleanly; the structure is identical, only the wrapper differs.

Keep it **concise**: terse tables, short bullets, no narrative padding.

```
# <feature name> - Documentation overview & coverage
_Generated <YYYY-MM-DD HH:MM UTC> by `documentation-mapper` (mode: set)._

Source: <one line — N pages / CQL / space, ≈ <total_chars> chars>.
> Labels: <feature-slug> · overview · cursor          # if label fallback used

## The N drafts
| # | Template          | Doc kind   | Title       | Page ID | Status        | URL | Missing% | Excess% |
|---|-------------------|------------|-------------|---------|---------------|-----|----------|---------|
| 1 | <template title>  | functional | <title 1>   | <id 1>  | draft/current | <link with ?draftShareId=… if draft> | <m1>% | <e1>% |
| ...                                                                                                       |

(Optional: one-line summary per draft — skip if the table already says it.)

## Aggregate coverage (union of the N drafts vs. source)
| Bucket | Chars | % of source |
|---|---|---|
| Covered by ≥1 draft       | <unique_covered_chars> | <aggregate_coverage_pct>% |
| Not covered by any draft  | <not_covered_chars>    | <not_covered_pct>%        |

Residual breakdown (one bullet per source page).

## Reference map (replaces the redundancy section)

Because each source block has exactly one owner across the set, body
content is never duplicated — drafts cross-reference each other via
`excerpt-include` macros. The interesting set-level metric is which
drafts pull excerpts from which others.

| Draft                       | Owned chars (≈) | Pulled-in excerpt chars (≈) | Reads excerpts from              |
|-----------------------------|------------------|-----------------------------|----------------------------------|
| <Draft 1>                   | <owned_chars(0)> | <reference_chars_into(0)>   | <Draft 3 (≈X chars), Draft 2 (≈Y chars)> |
| ...                                                                                                            |

About <dedup_ratio_pct>% of the body content would have been duplicated
if drafts had been emitted independently — the single-owner rule absorbed
that into <references_count> cross-reference macros.

### Pairwise reference flow

One row per non-zero pair, sorted by chars pulled. `a → b` reads as
"draft b includes excerpts owned by draft a".

| Flow         | ≈ chars pulled                | What is being pulled            |
|--------------|-------------------------------|---------------------------------|
| D_a → D_b    | <references_chars(a, b)>      | <grouped owner section labels>  |
| ...                                                                            |

## Per-source-page perspective
| Source page           | Coverage              | Block owner spread                          |
|-----------------------|-----------------------|---------------------------------------------|
| <link to source page> | <page_coverage_pct(p)>%| owned by <D_a (X chars), D_b (Y chars), ...> |
| ...                                                                                                 |

## Per-template comparison
| Template     | Sections     | Missing% | Excess% | Absorbs     | Discards    |
|--------------|--------------|----------|---------|-------------|-------------|
| <Template 1> | <n_sections> | <m1>%    | <e1>%   | <one phrase>| <one phrase>|
| ...                                                                              |

## Reading the numbers
- <high dedup_ratio = drafts are tightly coupled — consider merging or
  adding a shared "concepts" doc>
- <near-zero dedup_ratio = drafts are independent — good for parallel
  reading>
- <call out child drafts whose reference_chars_into dwarfs their owned
  body — that draft is closer to a "guided reading view of the parent"
  than a standalone document. Under parent-priority this is expected,
  not a smell; flag only if the operator wants to flatten the set.>

## Caveats
- Directional numbers (placements map, not a re-fetch + diff).
- References are strictly backward (child → parent) and resolve at the
  moment the child page is published, because Step 10 publishes drafts
  in reading-order rank with the owner first. No "page not found"
  caveat applies under the current rule.
- <label-fallback note if labels could not be applied natively>

## Follow-ups
1. Apply native labels on all pages.
2. Promote drafts to `current` in publish order (rank-1 first) so reader
   navigation matches the reading order.
3. Review any child draft whose `reference%` is so high that the draft
   adds no narrative of its own — the operator may want to fold it into
   the parent or drop it altogether. Under parent-priority this is a
   normal outcome for narrowly-scoped templates, not a defect.
4. Investigate any block whose owner ranks high (deep in the reading
   order) and that no parent draft has a fit for — the operator may
   want to add a matching section in the parent template so the block
   moves up the hierarchy on the next run.
```

### Worked example — Linked Audience US (3-template set)

> **Pre-parent-priority run.** The numbers below come from a set-mode
> run that pre-dates BOTH the single-owner rule and the parent-priority
> ownership rule (now in § Set-mode de-duplication via reading-order
> references). The original drafts duplicated body content across
> templates and assigned owners by best semantic fit regardless of
> rank, which produced cycles and forward (parent → child) references.
> A modern re-run of the same input would yield:
>
> - ≈ 0 % redundancy (single-owner rule), with the duplicated share
>   surfacing as `excerpt-include` traffic in the reference map
>   instead.
> - **No cross-rank ownership**: the rank-1 (parent) draft owns every
>   block for which its template has a matching section, so rank-2 and
>   rank-3 drafts become reference-heavy reading views. `reference%`
>   on the child drafts is high by design.
> - **Acyclic reference graph**: strictly backward (rank-N → rank-M
>   where M < N). No forward references, no prose pointers.

A real run of set mode against `Linked Audience US` (10 HOV source pages +
1 inline-comment thread, ~61,500 chars) using OVSD's three feature
templates produced:

| Draft | Sections | Excess% (single-doc view) | Footprint (≈ chars) |
|---|---|---|---|
| Linked Audience US - Functional documentation | 8 | ~58% | ~17,000 |
| Linked Audience US - Solution / Design | 11 | ~20% | ~45,000 |
| Linked Audience US - Runbook / Operations | 7 | ~59% | ~18,000 |
| **Sum of per-draft appearances** | | | **~80,000** |
| **Unique source content covered (union)** | | | **~57,800** |
| **Duplicated content (sum − union)** | | | **~22,200** |

→ **Aggregate coverage ≈ 94%** (57,800 / 61,500) and
**redundancy ≈ 28%** (22,200 / 80,000), with the largest pairwise overlap
(Solution/Design ∩ Runbook ≈ 18,500 chars) carrying ~83% of all
duplication. Under the new single-owner rule, that ~22,200-char
duplication would instead surface as `excerpt-include` traffic in the
reference map (`Solution/Design → Runbook` ≈ 18,500 chars, plus residual
flows) with a `dedup_ratio_pct` near 28%.

The four published pages live in OVSD under the `PoC LLM agent` folder
(`66947547990`):

- `66948596394` — Functional documentation (draft)
- `66947548074` — Solution / Design (draft)
- `66947581010` — Runbook / Operations (draft)
- `66948498208` — Documentation overview & coverage (draft, the Step-12
  product of this same set-mode run)

Use it as a historical reference; do not expect a fresh run to reproduce
the redundancy figure.
