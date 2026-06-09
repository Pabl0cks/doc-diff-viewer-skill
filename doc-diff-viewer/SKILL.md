---
name: doc-diff-viewer
description: Given two versions of a document (original and final/edited), generate a single self-contained HTML "track changes" viewer with word-level inline diffs — deletions in red strikethrough, additions in green — and a Track changes / Original / Final view toggle. Use when the user asks to compare two docs or drafts, show what changed between versions, or build a diff/redline view of an article.
---

# Document Diff Viewer (inline track-changes HTML)

Generate ONE self-contained HTML file that shows the differences between two
versions of a document, inline, like Word / Google Docs "suggesting" mode.
Do NOT dump the HTML code in chat — save the file and give a 2–3 line summary
of the major edits with a link to the file.

## Step 1 — Read and map

Read BOTH documents fully. Walk them chronologically and map every difference.
Categories:

- **Modified**: fragment exists in both but words/punctuation changed → `<del>old</del><ins>new</ins>`
- **Deleted**: exists only in the original → `<del>old</del>`
- **Added**: exists only in the final → `<ins>new</ins>`
- **Unchanged**: plain text, no markup.

## Step 2 — Granularity rules (CRITICAL — the user wants surgical diffs)

- Highlight ONLY the words that actually changed. If the edit is 1–2 words or
  punctuation, mark just that fragment — NEVER wrap the whole sentence.
  - `with` → `to` becomes: `system <del>with</del><ins>to</ins> markdown`
  - comma added: attach to the neighboring word so both views read correctly:
    `<del>check since</del><ins>check, since</ins>` (or `manually<ins>,</ins> and` for a pure insertion)
- Use whole-sentence/paragraph `<del>...</del><ins>...</ins>` ONLY for genuine
  rewrites where word-level mapping would be noise.
- Pure word additions (e.g. "Opus" → "Opus 4.6"): `Opus<ins> 4.6</ins>` — no del.
- A sentence split into its own paragraph: mark with `<ins class="para-break" title="paragraph break added">¶</ins>`
  (visible only in the redline view).
- Removed image embeds: `<del class="img-del removed-when-final">![alt](src)</del>` as a block.
- Markdown link replaced by inline code (e.g. broken `http://SKILL.md` links):
  wrap just that element: `<del><a href="...">SKILL.md</a></del><ins><code>SKILL.md</code></ins>`.
- Heading edits: put del/ins inside the `<h2>/<h3>`, e.g.
  `<h3>Why the failures are systematic<del>, not random</del></h3>`.
- Typo fixes (double periods, misspellings) are Modified fragments like any other.

## Step 3 — Invariant: three readable views

The file has a body class toggle: `view-redline` (default), `view-original`,
`view-final`. The markup MUST satisfy:

- Hiding all `<ins>` and unstyling all `<del>` reproduces the ORIGINAL text exactly.
- Hiding all `<del>` and unstyling all `<ins>` reproduces the FINAL text exactly.

So `<del>` content = exact original fragment (with its formatting: links, code,
bold), `<ins>` content = exact final fragment. Watch spaces at fragment
boundaries — read each rendered sentence in both views mentally to verify.

## Step 4 — HTML template

Single file: embedded CSS + vanilla JS, article centered at max-width 800px,
clean typography, rendered markdown (headings, tables, lists, links, `code`).
Sticky toolbar with the three view buttons and a mini legend.

Skeleton (reuse verbatim, put the diffed article inside `.container`):

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Change Review — {doc title}</title>
<style>
  body { font-family: -apple-system, "Segoe UI", Roboto, sans-serif; line-height:1.7;
         color:#37352f; background:#fbfbfa; margin:0; padding:40px 20px 120px; }
  .container { max-width:800px; margin:0 auto; }
  h2 { border-bottom:1px solid #e9e7e2; padding-bottom:6px; margin-top:1.8em; }
  blockquote { border-left:3px solid #37352f; padding:4px 16px; color:#6b6960; font-style:italic; }
  code { font-family:Consolas,Menlo,monospace; background:#f1f0ec; color:#c0392b;
         padding:1px 5px; border-radius:4px; font-size:.88em; }
  a { color:#2563eb; text-decoration:none; } a:hover { text-decoration:underline; }
  table { border-collapse:collapse; width:100%; margin:1.2em 0; font-size:.92em; }
  th,td { border:1px solid #e0ded8; padding:7px 11px; text-align:left; }
  th { background:#f4f3ef; } tr:nth-child(even) td { background:#fafaf8; }

  .toolbar { position:sticky; top:0; z-index:50; background:#fbfbfa;
             border-bottom:1px solid #e9e7e2; padding:12px 4px; margin-bottom:28px;
             font-size:.85em; display:flex; gap:16px; align-items:center; flex-wrap:wrap; }
  .toolbar button { font:inherit; padding:4px 14px; border:1px solid #d9d6cf;
                    background:#fff; border-radius:6px; cursor:pointer; color:#57544c; }
  .toolbar button.active { background:#37352f; color:#fff; border-color:#37352f; }
  .chip-del { background:#ffe3e3; color:#c92a2a; text-decoration:line-through; padding:1px 8px; border-radius:4px; }
  .chip-ins { background:#d3f9d8; color:#2b8a3e; padding:1px 8px; border-radius:4px; }
  .toolbar .hint { color:#908d85; margin-left:auto; }

  .container del { background:#ffe3e3; color:#c92a2a; text-decoration:line-through;
                   text-decoration-color:#e03131; border-radius:2px; padding:0 2px; }
  .container del code { background:#ffd6d6; color:#c92a2a; }
  .container del a { color:#c92a2a; }
  .container ins { background:#d3f9d8; color:#1d6b32; text-decoration:none;
                   border-radius:2px; padding:0 2px; }
  .container ins code { background:#b9ecc4; color:#1d6b32; }
  del.img-del { display:block; font-family:monospace; font-size:.85em; padding:8px 12px; margin:1em 0; }
  ins.para-break { font-weight:700; padding:0 6px; }

  body.view-original .container ins { display:none; }
  body.view-original .container del { background:none; color:inherit; text-decoration:none; padding:0; }
  body.view-original .container del code { background:#f1f0ec; color:#c0392b; }
  body.view-original .container del a { color:#2563eb; }

  body.view-final .container del { display:none; }
  body.view-final .container ins { background:none; color:inherit; padding:0; }
  body.view-final .container ins code { background:#f1f0ec; color:#c0392b; }
  body.view-final ins.para-break { display:none; }
  body.view-final .removed-when-final { display:none; }
</style>
</head>
<body class="view-redline">
<div class="toolbar">
  <button data-view="view-redline" class="active">Track changes</button>
  <button data-view="view-original">Original</button>
  <button data-view="view-final">Final</button>
  <span><span class="chip-del">deleted</span> <span class="chip-ins">added</span></span>
  <span class="hint">Red strikethrough = removed from the original · Green = what replaced it</span>
</div>
<div class="container">
  <!-- diffed article here -->
</div>
<script>
document.querySelectorAll('.toolbar button[data-view]').forEach(function (btn) {
  btn.addEventListener('click', function () {
    document.body.className = btn.getAttribute('data-view');
    document.querySelectorAll('.toolbar button').forEach(function (b) { b.classList.remove('active'); });
    btn.classList.add('active');
  });
});
</script>
</body>
</html>
```

## Step 5 — Output & maintenance

- Save as `article-changes-{slug}.html` next to the source docs (or in the
  folder the user names). Reply with a 2–3 line summary of the major edits and
  a link to the file — never paste the full HTML in chat.
- If the user later edits one of the source documents, RE-READ the changed file
  (don't trust memory of it), recompute only the affected fragments, and patch
  the HTML with targeted edits. After patching, re-verify the three-views
  invariant for the touched sentences.
