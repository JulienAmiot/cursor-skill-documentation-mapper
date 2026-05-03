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
  and overall). Use when the user asks to map, migrate, consolidate, transcribe
  or rewrite documentation into a Confluence template, mentions
  "documentation mapper", "doc mapping", "fill a Confluence template from
  existing docs", or wants a Confluence-to-Confluence proof of concept of
  template-driven documentation.
---

# Documentation Mapper

Maps documentation from arbitrary sources into a target template, footnotes
every block with its source URL, and quantifies what got lost (empty target
sections) and what got rerouted (source content that did not fit anywhere in
the template).

The default proof of concept is **Confluence → Confluence**.

## Workflow

Copy this checklist and update it as you go:

```
- [ ] Step 1: Gather operator inputs
- [ ] Step 2: Discover the Atlassian MCP server in the current project
- [ ] Step 3: Fetch and split the source(s) into blocks
- [ ] Step 4: Fetch and parse the target template structure
- [ ] Step 5: Map each source block to a target section
- [ ] Step 6: Add a source-URL footnote to every mapped block
- [ ] Step 7: Mark empty target sections as "content missing"
- [ ] Step 8: Build the "Discrepancies" section from excess content
- [ ] Step 9: Append the coverage footnote (missing% + excess%)
- [ ] Step 10: Publish or preview at the destination
```

## Step 1 — Gather operator inputs

Use the `AskQuestion` tool when available, otherwise ask conversationally.
Always ask, even when context suggests an obvious answer:

1. **Source type** — default `Confluence`. Offer at least: `Confluence`,
   `Jira`, `Markdown / local files`, `PDF`, `Web URLs`, `Mixed`.
2. **Source location** — for Confluence, accept any of:
   - one or more page IDs,
   - a parent page ID (descendants will be pulled),
   - a CQL query (e.g. `space = ENG AND label = onboarding`),
   - a space key (use with care — can be huge).
3. **Template** — a Confluence page ID OR a Confluence template inside a
   space. The structure (heading tree) AND any example content under each
   heading are both inputs to the mapper.
4. **Destination** — either a new page (`parent page ID` + `title`) or an
   existing page to update. Always confirm the destination space.

Record every answer back to the operator before doing any work.

## Step 2 — Discover the Atlassian MCP server

The Atlassian MCP server identifier varies per project. Discover it before
calling any tool. From the workspace root:

```
ls mcps/*/tools/getConfluencePage.json
```

The parent directory of the match is the server id (for this repository it is
`project-0-tree-graph-viz-atlassian`). Use that id in every `CallMcpTool` call.
If no match is found, tell the operator the project has no Atlassian MCP
configured and stop.

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

## Step 4 — Fetch and parse the target template

Call `getConfluencePage` on the template id. Build the **target tree**:

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

## Step 5 — Map each source block to a target section

For every source block, score every target section on:

- semantic similarity of the block's heading to the target heading,
- semantic similarity of the block's body to the target's example content.

Pick the highest-scoring section, breaking ties by depth (deepest wins —
specific sections beat generic parents).

If the best score is below a confidence threshold, **do not force the match**.
Mark the block as `excess` and tag it with a short `topic` derived from its
heading. Excess blocks go to the Discrepancies section in Step 8.

## Step 6 — Source-URL footnotes

Every block placed into the final document MUST carry a footnote pointing at
its `source_url`. In Confluence storage format, prefer the footnote macro when
available; otherwise use inline superscript markers (`[1]`, `[2]`, …) and a
`References` block at the end of each section listing the URLs in order.

Never silently merge two blocks from different sources without preserving both
URLs as distinct footnotes.

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
Phrasing stays neutral: this section relocates content, it does not comment on
it.

## Step 9 — Coverage footnote

Append a final footnote at the very end of the document, titled
`Mapping coverage`. It MUST contain BOTH metrics, computed as follows.

**Missing% (section-count basis)**

```
missing% = empty_target_sections / total_target_sections * 100
```

Phrase it like the operator's own examples — for instance, "10% of the
information is missing (1 of 10 target sections empty)". Then list the empty
sections by name.

**Excess% (character-count basis, per topic and overall)**

```
excess%(topic) = excess_chars(topic)
                 / (excess_chars(topic) + retained_chars(topic))
                 * 100

excess%(overall) = sum(excess_chars) / (sum(excess_chars) + sum(retained_chars)) * 100
```

Phrase each topic line like the operator's own example — for instance, "66% of
the documentation on topic X does not make it into the final document" (which
is what you get when excess is twice the retained volume). Then list every
excess block with its source URL.

The two metrics are deliberately not symmetric: missing% is measured by
**section count** (structure coverage), excess% is measured by **character
count** (volume of source content that did not survive the mapping). Both must
appear.

See [reference.md](reference.md) for a full worked example.

## Step 10 — Publish

- New page → `createConfluencePage` under the chosen parent.
- Existing page → `updateConfluencePage`.

Print the resulting page URL back to the operator before exiting.

## Additional resources

- [reference.md](reference.md) — Atlassian MCP tool index, Confluence storage
  format notes, AskQuestion templates, and a fully worked metrics example.
