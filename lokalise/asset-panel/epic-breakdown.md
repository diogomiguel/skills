# Asset Panel V1 — Full RFC snapshot

Snapshot of the internal Confluence page "Asset Panel V1 - Work Breakdown". Re-fetch via `mcp__atlassian__getConfluencePage` if this is stale.

## Context

Reviewers and linguists in Vantage have no in-editor access to Translation Memory (TM) or Glossary. They can't fuzzy-search TM, insert previous translations, or look up glossary terms in context. Every competing CAT tool offers this. The Asset Panel adds TM + Glossary to the editor sidebar, surfacing fuzzy matches and terminology in context per segment.

## Refinement Meeting Decisions

- **Assets are workspace-scoped.** Currently assets are stored at workspace level and it's complex to count and search within assets used in a project (in practice impossible with current model considering data volume and latency). Therefore: search works within workspace scope, and project-level counters are skipped for now. Once assets become project-scoped it will be adjusted accordingly.
- **No TM match pagination.** Pagination is complex because of the custom matching logic. Currently returns at most 10 entries without pagination. Skip pagination but adjust the upper limit (currently 10, can be 20–30). Check upstream for the actual return count to apply the proper limit.
- **Smart TM Insert with preserving NTCs is a V1 priority.** Important functionality, needs BE research, most complex topic.
- **Glossary term filtering — unblocked.** Filters target language only, AND operator stacking. Same approach as existing "TM approved" filter.
- **Lupo UI migrations are separate work** — tracked under a separate epic, not part of Asset Panel V1.

## Current State (already exists before V1)

| What exists | Status |
|---|---|
| Editor sidebar with simple "Assets" header and open/close toggle | Works |
| Content/Assets mode switching via `ap_feature_assets_panel` flag | Works |
| Glossary term list — accordion with matched terms, count badge, expand per term, rules display, "Add term" dialog | Works |
| TM match fetching — bulk batched per segment, returns fuzzy matches | Works |
| TM inline highlights in segment text with hoverable popover | Works |
| Segment editing — mutation to update translation text | Works |
| Non-translatable content (NTC) handling — parse, display, edit code tags | Mature |
| Per-segment context providing both TM matches and glossary terms | Works |

**Key gap:** TM matches are fetched but never shown in the sidebar. No insert functionality exists for either TM or glossary.

## Story 1 — Assets Panel Shell & Default Browse State (EXP-1747)

Panel header with Content/Assets tab bar, search icon, global assets page button (file-arrow-up — navigates to global assets page), close button. Switching tabs toggles content/assets modes.

Default state (no segment selected): guidance text prompting user to select a segment or search. "Search project assets" is a clickable blue link (with search icon) that opens the search input — second entry point beyond the header icon.

"Used in this workspace" section (Figma says "project" — use "workspace" per RFC) showing glossary + TM entry counts. Workspace-level data.

Wire TM matches + segment selection state through to the sidebar (currently only glossary terms are passed).

Panel opens by default when entering the editor. Fading gradient at bottom of scrollable areas.

**BE:** workspace-level asset counts API (if not already available). No project-level counting for V1.

Subtasks:
- EXP-1764 — Panel header shell (merged)
- EXP-1765 — Default browse state
- EXP-1767 — Fading scroll gradient

## Story 2 — TM Matches for Selected Segment (EXP-1748)

Collapsible "Translation Memory" section with match count badge, matches sorted by quality (101% context → 100% perfect → text → fuzzy, descending). Each match: type label with percentage, color-coded indicator (blue gradient for high matches, grey for fuzzy), truncated source + target.

Expandable detail, two variants:
- **Not applied:** full source/target, metadata (date, user avatar, name), full-width "Insert entry to segment" button at bottom
- **Applied (101% context):** previous segment context, source/target, next segment context, metadata — NO Insert button. Sections separated by dividers.

Hover diff preview: hovering a non-applied TM entry in the sidebar shows inline diff in the target segment — old text strikethrough, new text inline. Nothing changes until user clicks Insert.

Code tags / NTC render as formatted badges (same as segment editor). "Applied" badge when target text matches current segment translation. Fading gradient at bottom of list. Empty state: "No matches found. Try searching manually."

No pagination — cap raised from 10 to ~20–30.

**BE:** raise `maxMatches` cap to 20–30 (confirm upstream's actual return count).

## Story 3 — Glossary Terms for Selected Segment (EXP-1749)

"Glossary" section with matched term count badge and "Add term" (+) button. Each term: name, rules icons (non-translatable, case-sensitive), expand chevron. Expanded: description, rule details with "Applied to" translations per target locale (with locale badges), "See segments with this glossary term" button. "Insert" action to insert the term translation.

Bidirectional highlighting: hovering a term in sidebar highlights it in editor. Glossary highlights in editor switch to dotted underline (avoid conflict with upcoming comment highlighting).

Remove "Reapply glossary" stub button (not in V1). Empty state when no terms matched.

## Story 4 — Insert TM Match into Segment (EXP-1750)

Full-width "Insert entry to segment" button at bottom of expanded (not-applied) TM match detail. Click replaces segment's translation with the TM match target, with loading state and success/error feedback. Applied matches don't show the Insert button.

After successful insert: "Applied" badge updates. For perfect/context matches with matching NTC: straight replacement. For fuzzy matches with differing NTC: defer to Smart Insert (Story 5).

Amplitude event: `tm_match_inserted` with match type, match %, segment ID.

## Story 5 — Smart Insert, NTC/Tag Validation for Fuzzy Matches (EXP-1751) — V1 priority

When inserting a fuzzy TM match with different code tags / NTC than the target segment, validate alignment and prevent silent corruption of exports.

**Why it matters:** Fuzzy matches often have different tag structures. Inserting blindly can corrupt DOCX/PPTX/HTML exports — this has been reported by customers.

**Approach:** Experimental LLM-based "smart insert" to reconstruct code tags. Only engages when code tags detected.

FE:
- Before insert of fuzzy match, detect whether either side has code tags (OKAPI `{op:N}`, `{cl:N}`, `{ph:N}`, or HTML tags)
- If none or matching: straight insert. If differ: confirmation dialog with "Insert with tag adjustment" (if LLM available), "Insert as-is", or "Cancel"
- **NTC variant switcher** in the modal — appears only when more than 1 NTC variant.
- Fallback: if LLM unavailable, still allow insert with warning

BE spike: LLM-based tag reconstruction endpoint — accepts original segment value + TM match value, returns reconstructed text with tags repositioned. Never adds new code, only rearranges existing tags.

**BE concern:** No NTC/tag validation or reconstruction endpoint exists. The `generate-variants` endpoint uses Polyglot SDK for rephrasing/shortening — not tag repositioning. New endpoint needed. FE can ship detection + warning dialog without BE; LLM reconstruction can be a fast-follow.

## Story 6 — Glossary Term Insertion via Slash Command (EXP-1752)

Typing `/` in segment editor triggers glossary term dropdown inline at cursor. All glossary terms listed (not just matched). First item auto-selected, its detail popover appears alongside the dropdown.

As user types after `/`, dropdown narrows (search/filter). Popover updates to active item.

If no term matches → "Add new glossary term" suggestion — opens Add Term modal with typed text pre-filled.

Exit with **double space** — dismisses slash command, leaves text as plain text (Slack/Notion pattern).

Handle multiple translations per term (picker), edge cases (no terms available, cursor at NTC boundary, term already present). Extensible for future commands.

Amplitude event: `glossary_term_inserted` with term ID + method (panel vs slash).

Subtasks: EXP-1776, EXP-1777.

## Story 7 — Free-Text Asset Search (EXP-1753)

Search icon or "Search project assets" link opens search input replacing the tab area. Debounced (300ms), queries both TM and Glossary, results grouped by type.

**Search term highlighting**: matching text wrapped with rounded yellow background overlay.

Context-aware:
- **No segment selected:** searches whole workspace — glossary terms + TM entries. Does NOT show fuzzy matches.
- **Segment selected:** glossary search goes through whole glossary; TM search filters within already-loaded fuzzy matches (client-side, no new API call). Does NOT search whole TM.

Loading, empty, error states.

BE: both `GET /v1/owners/:ownerId/records` (TM, cursor-paginated) and `GET /v1/workspaces/:workspaceId/glossary/terms` (glossary) use `flex-search-query` — both workspace-scoped, acceptable for V1.

Subtasks:
- EXP-1778 — Search mode UI (merged)
- EXP-1779 — Context-aware search (workspace vs fuzzy match filtering)

## Story 8 — Glossary Term as Segment Filter (EXP-1754)

"See segments with this glossary term" action on each glossary item in the expanded view. When already applied, button is **hidden** (not disabled — completely removed).

Click applies a target-text filter with AND logic on top of existing filters. Applied filter shows in the filters bar and can be cleared.

**Filter searches target language only.** If reviewer chose not to use the term in their translation, that segment won't appear. Goal: show where terms were actually applied, not where they could have been. Same approach as "TM approved" filter.

Example stacking:
- Quality = Major + glossary term "API" → segments with Major quality AND term "API"
- Plus search "example text" → all three stacking

**BE concern:** Editor content filter API doesn't support filtering by glossary term presence. Current filter dimensions: status, language, category, task, LQA severity, flex-search. Two options:
- (a) Add `filter-glossary-term` param to editor content API (BE work)
- (b) Client-side cross-referencing with loaded segments (limited to current page, won't scale)

## Out of V1 scope

| Feature | Reason |
|---|---|
| TM entry editing from panel | Post-V1 — asset page handles edits |
| Glossary entry creation from panel | Post-V1 (basic "Add term" dialog stays) |
| Bulk TM insert (multi-segment) | Post-V1 — V1 is per-segment only |
| Multiple TM/Glossary per project | Blocked on Translations domain shared assets initiative |
| TM analysis files/reports | Post-V1 — separate LSP billing feature |
| Lupo UI migrations | Separate epic |
| Segment comments | Separate initiative |
| Segment history / track changes | Separate initiative |
