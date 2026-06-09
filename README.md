# doc-diff-viewer

A Claude Code skill that compares two versions of a document (original and
final/edited) and generates a single self-contained HTML "track changes"
viewer: word-level inline diffs with deletions in red strikethrough and
additions in green, plus a **Track changes / Original / Final** view toggle.

**[▶ Live demo](https://pabl0cks.github.io/doc-diff-viewer-skill/evaluating-skills-article/article-changes-evaluating-agents.html)** — the generated viewer for the example article below.

## Contents

- [`doc-diff-viewer/SKILL.md`](doc-diff-viewer/SKILL.md) — the skill. To install it for all
  your projects, copy the `doc-diff-viewer` folder to `~/.claude/skills/`.
- [`evaluating-skills-article/`](evaluating-skills-article/) — first real usage of the skill:
  - `New-Evaluating agents skills testing whether they are_original.md` — original draft
  - `new-evaluating-agent-skills-final-opus.md` — final edited version
  - `article-changes-evaluating-agents.html` — the generated diff viewer (open in a browser)

## Usage

With the skill installed, give Claude Code two document versions and ask
something like:

> Show me the differences between `draft.md` and `final.md` like a
> track-changes view.

The result is one HTML file saved next to the sources — no dependencies,
works offline.
