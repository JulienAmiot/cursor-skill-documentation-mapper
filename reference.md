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

Step 1 can be driven by five AskQuestion blocks:

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

3. "Where should the new document land?"
   - New page under <parent page ID>  (title will be inferred in Step 10
     from "<feature name> - <template title>" and confirmed before publish)
   - Update existing page <page ID>   (title left untouched)

4. "Is this a technical or functional document?"
   - technical
   - functional

5. "What is the feature name?" (free text; propose a default if you can
   infer it from the chat — e.g. operator said "document the auth refresh
   flow" → propose "auth refresh flow")
```

Always echo the operator's choices back before running anything destructive.

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
