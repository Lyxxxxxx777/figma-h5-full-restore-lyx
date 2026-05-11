# figma-h5-full-restore-lyx

> End-to-end pipeline for restoring an H5 activity page **1:1** from a Figma link.

An [Agent Skill](https://www.anthropic.com/engineering/agent-skills) that orchestrates Figma asset export, page implementation, and local visual self-check into a single workflow. Built on top of the `figma-implement-design` and `prd-implementation-precheck` skills, and powered by the **Vibma MCP** (for Figma reads/exports) and **Chrome DevTools MCP** (for visual verification).

---

## When to use

✅ Use this skill when:
- You have a Figma link **and** want to restore a full H5 activity page from it
- You want a full pipeline: cut assets → implement code → visually verify against the design

❌ Do **not** use this skill when:
- You only need to restore a single component → use [`figma-implement-design`](https://github.com/anthropics/skills) instead
- You're doing pure PRD work or bug fixing (no Figma restoration) → use `prd-implementation-precheck` or `verification`
- You want to **author** Figma nodes from the agent → use `figma-use`

---

## Core mental model

> **Figma is source code, not a reference image.**

Before any visual decision, the agent must answer three questions:

1. **What's the node ID in Figma?** → call `Vibma.frames.get` to read structure, never guess from a flattened render
2. **What's the `absoluteBoundingBox`?** → this is the truth for CSS positioning, never use `flex justify-content: center` by feel
3. **Are there sibling nodes with `visible: false`?** → tab on/off states, button pressed states must be exported in one pass

In real-world stats, pages that skip the "survey-first" flow average **6–7 rework rounds**; pages that follow the full flow converge in **≤3 rounds**.

---

## Three-step pipeline

| Step | Goal | Key tool |
|------|------|----------|
| **1. Cut** | Export every needed asset from Figma (full pages + components + multi-state) | Vibma MCP |
| **2. Build** | Write the page code following project conventions; cross-check the PRD if provided | `figma-implement-design` + `prd-implementation-precheck` |
| **3. Verify** | Screenshot the running page, compare against Figma, iterate until visually aligned | Chrome DevTools MCP |

See [`SKILL.md`](./SKILL.md) for the full specification.

---

## Installation

### Recommended: via the [skills.sh](https://skills.sh) CLI

```bash
npx skills add Lyxxxxxx777/figma-h5-full-restore-lyx
```

This installs the skill into your agent's skills directory automatically and contributes anonymous telemetry to the [skills.sh leaderboard](https://skills.sh).

### Alternative: manual clone

```bash
# macOS / Linux
cd ~/.agents/skills
git clone https://github.com/Lyxxxxxx777/figma-h5-full-restore-lyx.git

# Windows (PowerShell)
cd $env:USERPROFILE\.agents\skills
git clone https://github.com/Lyxxxxxx777/figma-h5-full-restore-lyx.git
```

After installing, restart your agent client (CodeMaker / Claude Code / Cursor / etc.) so it picks up the new skill.

---

## Required dependencies

This skill orchestrates the following — make sure they are available in your agent before running:

| Type | Name | Purpose |
|------|------|---------|
| Skill | `figma-implement-design` | Standard Figma → code workflow |
| Skill | `prd-implementation-precheck` | PRD precheck and risk surfacing |
| MCP | **Vibma** | Reading Figma node trees and exporting assets |
| MCP | **Chrome DevTools** | Local visual verification (screenshot + console) |

If any are missing, the skill will halt and prompt you to install them — it does **not** attempt to install MCPs/Skills on your behalf.

---

## Required inputs

When invoking, supply at minimum:

1. **Figma link** (required) — must point to a specific frame, e.g. `https://figma.com/design/<fileKey>/<name>?node-id=1-2`
2. **Scope** (optional, default = "first frame") — `first frame` | `all frames` | `frame list`
3. **PRD path** (optional, prompted if not provided) — agent will run a precheck report and ask for confirmation before coding

---

## Example invocation

> "Use this Figma link `https://figma.com/design/abc/xyz?node-id=1-464` to restore an H5 page. PRD is at `docs/prd-event-page.md`. Implement all frames."

The agent will:
1. Validate inputs and dependencies
2. Sniff project conventions (Vue/React, base width, asset path, alias)
3. Activate the two underlying skills
4. Run the full **Cut → Build → Verify** pipeline
5. Iterate the verification loop until the rendered page matches the Figma design

---

## Author

**lyx** — distilled from real-world H5 activity-page restoration workflows.

---

## License

[MIT](./LICENSE)
