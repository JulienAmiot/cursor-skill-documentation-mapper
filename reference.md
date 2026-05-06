# Documentation Mapper — Reference

Detailed material for the `documentation-mapper` skill. Read this when SKILL.md
points you here.

## Atlassian MCP tool index

Discover the server id with `ls mcps/*/tools/getConfluencePage.json` (the
parent directory name is the id; for this repo it is
`project-0-tree-graph-viz-atlassian`).

| Tool | Used in step | Purpose |
|------|--------------|---------|
| `getAccessibleAtlassianResources` | 2 | Sanity-check that the MCP can reach the operator's Atlassian site. |
| `getConfluenceSpaces` | 1 | Help the operator identify a space when they only know the name. |
| `getPagesInConfluenceSpace` | 3 | Bulk source enumeration when a whole space was requested. |
| `getConfluencePage` | 3, 4 | Fetch a single page (source OR template). Returns title, body, URL. |
| `getConfluencePageDescendants` | 3 | Walk a parent page's subtree as additional source. |
| `searchConfluenceUsingCql` | 3 | Pull sources by query (label, text, last-updated, etc.). |
| `getConfluencePageFooterComments` | 3 (optional) | Pick up review notes that hint at the canonical wording for a section. |
| `getConfluencePageInlineComments` | 3 (optional) | Same, scoped to a specific paragraph. |
| `createConfluencePage` | 10 | Publish a brand-new destination page. |
| `updateConfluencePage` | 10 | Overwrite an existing destination page. |

Always pass the discovered `serverIdentifier` to `CallMcpTool`; never hardcode
`project-0-tree-graph-viz-atlassian` in the skill output, because it is
project-specific.

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

5. "Pick the template(s)." (one in single mode, ≥1 in set mode)
   In set mode, first scan the destination space for pages with a
   `template` label (CQL: `space = <KEY> AND label = "template"`) or
   matching `*template*`/`{Name of the feature}*` titles, present them as
   "Title · Page ID · top-level sections (preview)", and let the operator
   multi-select.

6. "Doc kind for each template?" (asked once in single mode, once per
   template in set mode)
   - technical
   - functional
   Default per template title:
     - contains "functional"                     → functional
     - contains "runbook"|"operations"|"design"|
       "architecture"|"deployment"|"testing"     → technical
   Always offer the default and let the operator override.
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
In set mode, confirm the consolidated plan as a small table:

```
# · Template title                         · Page ID    · Doc kind  · Inferred title
1 · {Name of the feature} - Functional…    · 66947548021 · functional · Auth refresh - Functional documentation
2 · {Name of the feature} - Solution / D…  · 66948498160 · technical  · Auth refresh - Solution / Design
3 · {Name of the feature} - Runbook / Op…  · 66948596304 · technical  · Auth refresh - Runbook / Operations
```

### Title inference rule (Step 10)

```
destination_title = "<feature name> - <template title>"
```

- Use the operator's original casing for `<feature name>`, NOT the slug.
- Strip placeholder wrappers from the template title before composing:
  `[…]`, `<…>`, `{{…}}`, leading/trailing `Template` or `Modèle`.
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
- `createConfluencePage` and `updateConfluencePage` both accept storage-format
  bodies. Validate that every `<ac:*>` macro you emit is opened and closed.

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

2. **Inline argument on the publish tool.** Re-read
   `createConfluencePage.json` / `updateConfluencePage.json` and check for a
   `labels`, `tags`, or `metadata` property. If present, pass the three
   labels there and skip step 11's separate call.

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

### Per-block tracking schema

To compute redundancy in Step 12, each per-template run (Steps 5–9) MUST
update a shared `placements` map keyed by source-block id:

```
placements: {
  <block_id>: {
    "char_count":    <int>,                  # from Step 3
    "source_url":    "...",
    "drafts":        [                       # one entry per draft that placed
                                             # this block (excluding excess
                                             # placements in Discrepancies)
      { "draft_idx": <0..N-1>,
        "section":   "<target heading>",
        "kind":      "retained" | "excess" }
    ]
  }
}
```

Excess placements (blocks that landed in a `Discrepancies` section in some
draft) DO count toward that draft's footprint but DO NOT count toward "covered
by that draft" — they are covered only if some other draft placed them as
retained content. Mark them with `kind: "excess"` to keep the bookkeeping
clean.

Define:

- `is_covered_by(block, draft_idx)` ⇔ block has at least one entry with
  `draft_idx == draft_idx` and `kind == "retained"`.
- `appearances(block)` = number of drafts where `is_covered_by(block, *)` is
  true.

### Aggregate coverage formula (union)

```
total_chars            = sum(block.char_count for block in source_set)
unique_covered_chars   = sum(block.char_count for block in source_set
                             if appearances(block) >= 1)
not_covered_chars      = total_chars - unique_covered_chars

aggregate_coverage_pct = unique_covered_chars / total_chars * 100
not_covered_pct        = 100 - aggregate_coverage_pct
```

Also itemise `not_covered_chars` by source page so the operator sees what
the residual is (typically navigation scaffolding, decorative content, open
TODOs, comment metadata). This is what landed in §3 (Aggregate coverage) of
the overview page.

### Redundancy formulas

```
per_draft_footprint(d)   = sum(block.char_count for block in source_set
                               if is_covered_by(block, d))
sum_appearances          = sum(per_draft_footprint(d) for d in 0..N-1)
duplicated_chars         = sum_appearances - unique_covered_chars

redundancy_pct           = duplicated_chars / sum_appearances * 100
average_appearance_ratio = sum_appearances / unique_covered_chars
```

Pairwise overlap (for any pair of drafts `(a, b)`):

```
pairwise_overlap(a, b)   = sum(block.char_count for block in source_set
                               if is_covered_by(block, a)
                               and is_covered_by(block, b))
```

Triple overlap (only if N ≥ 3):

```
triple_overlap(a, b, c)  = sum(block.char_count for block in source_set
                               if is_covered_by(block, a)
                               and is_covered_by(block, b)
                               and is_covered_by(block, c))
```

For each non-trivial pairwise overlap (say > 5% of `sum_appearances`),
group the contributing blocks by `section` to label what is duplicated
(e.g. "Build procedure (D2 §7.2 vs D3 §4.1)"). This is what populates the
"Where the redundancy lives" table on the overview page.

### Per-source-page coverage

For each source page `p`, compute:

```
page_coverage_pct(p)     = sum(block.char_count for block in p
                               if appearances(block) >= 1)
                           / sum(block.char_count for block in p)
                           * 100

most_duplicated_pair(p)  = argmax_{(a,b)} sum(block.char_count for block in p
                                              if is_covered_by(block, a)
                                              and is_covered_by(block, b))
```

This populates the "Per-source-page perspective" table on the overview page.

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

## Redundancy across the drafts
| Draft                                  | Source chars used (≈) |
|----------------------------------------|-----------------------|
| <Draft 1>                              | <per_draft_footprint(0)> |
| ...                                    | ... |
| Sum of per-draft appearances           | <sum_appearances> |
| Unique source content covered (union)  | <unique_covered_chars> |
| Duplicated content (sum − union)       | <duplicated_chars> |

About <redundancy_pct>% of the combined footprint is redundant
(<average_appearance_ratio>× on average).

### Where the redundancy lives (pairwise + triple)
| Overlap   | ≈ chars                 | What's duplicated         |
|-----------|-------------------------|---------------------------|
| D_a ∩ D_b | <pairwise_overlap(a,b)> | <grouped section labels>  |
| ...                                                             |
| All N     | <triple_overlap(...)>   | <grouped section labels>  |

## Per-source-page perspective
| Source page          | Coverage              | Most duplicated across drafts? |
|----------------------|-----------------------|--------------------------------|
| <link to source page>| <page_coverage_pct(p)>%| <most_duplicated_pair(p)>     |
| ...                                                                             |

## Per-template comparison
| Template     | Sections     | Missing% | Excess% | Absorbs     | Discards    |
|--------------|--------------|----------|---------|-------------|-------------|
| <Template 1> | <n_sections> | <m1>%    | <e1>%   | <one phrase>| <one phrase>|
| ...                                                                              |

## Reading the numbers
- <which overlaps are healthy vs. collapsible>
- <call out any near-disjoint pair / outsized residual>

## Caveats
- Directional numbers (placements map, not a re-fetch + diff).
- <label-fallback note if labels could not be applied natively>

## Follow-ups
1. Apply native labels on all pages.
2. Promote drafts to `current` in dependency order so cross-links resolve.
3. Replace placeholder sibling links between drafts with real page links.
4. Collapse the largest pairwise overlap with `see <other doc> §x.y`
   cross-references (≈ <X> chars saved → redundancy drops to <projected>%).
```

### Worked example — Linked Audience US (3-template set)

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
duplication. The four published pages live in OVSD under the
`PoC LLM agent` folder (`66947547990`):

- `66948596394` — Functional documentation (draft)
- `66947548074` — Solution / Design (draft)
- `66947581010` — Runbook / Operations (draft)
- `66948498208` — Documentation overview & coverage (draft, the Step-12
  product of this same set-mode run)

Use it as the canonical reference when verifying set-mode output.
