# figma-audit

A [Claude Code](https://claude.com/claude-code) skill that audits a Figma file or Figma Slides deck for design quality issues and produces a ranked findings report — then offers to fix or annotate each one.

## What it checks

| | Category |
|---|---|
| A | Data inconsistencies across pages (formats, conflicting values, placeholder data) |
| B | Type sizing (one-off sizes, ramp drift, hierarchy violations) |
| C | Typos and grammar (reads every prose block, not just word lists) |
| D | Type formatting (weight, line height, tracking, case, color, family) |
| E | Alignment of similar elements, within and across frames |
| F | Design errors (clipping, stale/duplicate frames, detached instances, radius/color drift) |
| G | Jumping pixels — static elements shifting between sequential frames or slides |
| H | Claims & narrative integrity for case studies and pitch decks |

Findings are graded **0** (blocker, needs a decision), **1** (UX issue), **3** (fixable), **4** (polish).

## Requirements

- Claude Code
- The [Figma MCP server](https://help.figma.com/hc/en-us/articles/32132100833559) connected (uses `use_figma` and `get_screenshot`)

## Install

```bash
git clone https://github.com/destinielise/figma-audit.git
cp -r figma-audit/skills/figma-audit ~/.claude/skills/
```

## Use

In Claude Code, run `/figma-audit` or say something like "audit this Figma" and paste a Figma link.

## License

MIT
