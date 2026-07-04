---
name: auto-design
description: Use when given a Figma Section or Frame link (often via "/auto-design:auto-design <url>") whose sub-frames each contain a "Claude description" spec plus reference material, and you must produce design mockups for each feature using the user's own design system. Triggers on a figma.com/design link to a section of feature frames to design.
---

# auto-design — Design a Figma PRD section with your design system

## Overview
Turn a Figma "design agent" Section into finished mockups. Each sub-Frame is one feature: it holds a **"Claude description:"** text node (the spec — sometimes with reference node-ids/URLs) plus reference material (existing app screens, component examples). For each feature, produce new screens built from **the configured design system**, placed to the **right** of that feature's reference material.

**Converge-only:** build and refine **one** design direction per feature. Do **not** generate multiple alternative design directions or exploratory variations — produce the single best design and iterate on it.

**Input:** a `figma.com/design/<fileKey>/...?node-id=<id>` URL to a Section or Frame. Extract `fileKey` and `nodeId` (node-id `1-2` → `1:2`).

## Required setup
- **You MUST load the `figma-use` skill before any `use_figma` call** (prefer `/figma-use`; fallback MCP resource `skill://figma/figma-use/SKILL.md`). Batch-load all Figma MCP tools in one `ToolSearch` `select:` call.
- Work **incrementally**: build one screen, `get_screenshot` to verify, fix before the next. `use_figma` is atomic — a thrown error rolls back the entire script, so keep scripts focused and `loadFontAsync` everything before touching text.

## Step 0 — Design system setup (first run)
This skill builds from **your** design system's Figma library. It remembers your choice in a config file so you're only asked once.

1. **Read the config** at `~/.claude/auto-design/config.json`. If it exists and has a `designSystem` entry, use its `name`, `figmaLibraryKey`, and `fontFamily` for the rest of the run and skip to the workflow.
2. **If there is no config** (or the user says "reconfigure"):
   - Call `get_libraries` (Figma MCP) on the target file to list the libraries enabled in it, and show them to the user.
   - Ask the user which design system to use, and capture:
     - `name` — a human label (e.g. "Acme DS").
     - `figmaLibraryKey` — the library key (`lk-...`), chosen from the `get_libraries` result or pasted directly.
     - `fontFamily` (optional) — the design system's default text font; if unknown, leave it blank and fall back to the reference material's font.
   - **Write** `~/.claude/auto-design/config.json` (create the `~/.claude/auto-design/` directory if needed) so future runs skip this step:
     ```json
     { "designSystem": { "name": "Acme DS", "figmaLibraryKey": "lk-...", "fontFamily": "Inter" } }
     ```
3. The design system's **library must be added to the target Figma file** for its components to import. If imports fail with a missing-library error, tell the user to add the library to the file.

> To switch design systems later, delete `~/.claude/auto-design/config.json` or say "reconfigure".

## Workflow (per Section)
1. **Enumerate.** `get_metadata` on the Section to list feature sub-frames + bounds.
2. **Read the spec.** Identify the feature's spec by the **`Claude description:` label** (the recommended convention) or, failing that, the prominent description text near the frame's title. **Text color/styling is irrelevant — no red or otherwise-highlighted text is required.** Note any referenced node-ids/URLs; they can be **stale** — on an "invalid node" error, fall back to the reference frames physically inside the feature frame.
3. **Study references.** `get_screenshot` the reference screens/components so new work matches their visual language and the design system.
4. **Plan the screens (converge).** Produce the **single best** design for the feature — do not create alternative versions to choose between. Use as many screens as the *flow* needs: if the spec says "beginning to end," show the full flow across multiple sequential screens. Multiple screens for one flow is **not** divergence; multiple alternative versions of the same screen **is** — don't do that.
5. **Build.** Clone the closest relevant reference screen, then overlay/replace content using the configured design system's components + its font (from config, else the reference material's font). Place each screen to the **right** of the reference material (x beyond the rightmost reference child). Add a heading + per-screen captions.
6. **Verify.** Screenshot each screen; check alignment, clipping, overlaps; fix before moving on.
7. **Report.** Per feature: what you built, screen node-id deep links, and any judgment calls — when the design system has no equivalent, **converge on the closest design-system-native pattern** (or a minimal custom element) and note the decision.

## Design system component reference
Discover components at runtime from your configured library — don't assume fixed keys.

- **Import sets:** `figma.importComponentSetByKeyAsync(key)`, then `set.defaultVariant.createInstance()` + `setProperties({...})`.
- **Import icons / single components:** `importComponentByKeyAsync(key)` → `.createInstance()`; recolor by setting `fills` on the instance's non-instance geometry children.
- **Find components and their keys** with `search_design_system` scoped to your `figmaLibraryKey`. Recall is poor for fuzzy queries — prefer exact component/variant names.
- **Reuse** a component already instanced in the file by cloning it when that is easier than importing.
- **Font:** use `fontFamily` from config; if unset, match the reference material's font. Always `loadFontAsync` every style before editing text.

## Common mistakes
| Mistake | Fix |
|---|---|
| Custom auto-layout "circle"/pill collapses to a thin sliver | Set BOTH `primaryAxisSizingMode` and `counterAxisSizingMode` = `'FIXED'` before `resize()` |
| "unloaded font" error when cloning or editing text | `loadFontAsync` every style used before cloning/appending/editing; for existing text read its font via `getStyledTextSegments(['fontName'])` first |
| A floating overlay docks at the wrong height above a lower-pinned element (e.g. a composer/toolbar) | The visible content frame often clips higher than where the lower element actually sits. Dock overlays in the **outer** container at `y = anchorY − overlayHeight − 10`; inner content frames are frequently vertical auto-layout with some children pinned absolutely |
| New design overlaps the reference material | Place it to the right — x beyond the rightmost reference child |
| Can't position children of an auto-layout frame | Use `layoutPositioning='ABSOLUTE'` for floating overlays/intros; otherwise they join the flow |
| Editing a node inside a component instance fails | Set component properties via `setProperties`, or edit exposed sublayers only |
| A cloned reference row renders invisible | Reference list rows are often hidden variants; a clone inherits `visible:false`. Set `node.visible = true` after cloning |
| Need to add an item to a sidebar/list that's a component instance | `detachInstance()` first, then clone an existing row and `insertChild`. Node ids change after detach — re-find by text |
| Appending a new item to a bottom-aligned list (e.g. a chat or activity log) | Bottom-aligned lists are vertical auto-layout with `primaryAxisAlignItems: 'MAX'`; append your node as the last flow child and it docks at the bottom, scrolling earlier items up |
| `findOne`/`findAll` throws while walking up parents | Those methods don't exist on leaf nodes (e.g. TEXT) and Figma *throws* on missing properties. Guard with `'findAll' in node`, or start the walk from `node.parent` (a container) |
| Cloning a sub-element grabs only the inner text (no card background/icon) | Walk up to the outermost frame that has a visible SOLID fill (the card), not the first text group; clone that |

## When NOT to use
Single ad-hoc mockups, or code-to-design of an existing web page (use `figma-generate-design`). This skill requires a Figma library for your design system to be added to the target file (the skill asks for its name + key on first run).
