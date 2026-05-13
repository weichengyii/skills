# Markdown Compatibility Matrix — GitHub × JetBrains

Per-feature support across the two target renderers. **JN** = JetBrains IDE bundled Markdown plugin (intellij-markdown / flexmark backend), including the bundled Mermaid extension under Markdown Extensions. **JN+** = JetBrains with an optional Marketplace plugin (e.g. standalone Mermaid plugin, Markdown Navigator). **GH** = GitHub web (cmark-gfm + GitHub post-processing).

Legend:
- ✅ renders correctly
- ⚠️ renders but with caveats — see Notes
- ❌ does not render — appears as literal source or is stripped
- 🔌 only with a specific plugin

## Core CommonMark

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| ATX headings `# … ######` | ✅ | ✅ | ✅ | Anchor slug rules differ — see github-specific.md |
| Setext headings (`===` / `---` underline) | ✅ | ✅ | ✅ | Avoid — anchor handling more fragile |
| Paragraphs | ✅ | ✅ | ✅ | |
| Bold / italic / inline code | ✅ | ✅ | ✅ | |
| Block quotes `>` | ✅ | ✅ | ✅ | Single-level. Nested + GFM alerts collide on GH |
| Unordered list `-` `*` `+` | ✅ | ✅ | ✅ | Adapt to existing file's bullet |
| Ordered list `1.` | ✅ | ✅ | ✅ | First number sets start; subsequent numbers ignored |
| Fenced code block ` ``` ` | ✅ | ✅ | ✅ | Language id triggers highlight (GH) and language injection (JN) |
| Indented code block (4 spaces) | ✅ | ✅ | ✅ | Prefer fenced — no language id otherwise |
| Inline link `[t](url)` | ✅ | ✅ | ✅ | |
| Reference link `[t][r]` + `[r]: url` | ✅ | ✅ | ✅ | |
| Autolink `<https://…>` | ✅ | ✅ | ✅ | |
| Bare-URL autolink (no angles) | ✅ | ⚠️ | ✅ | GH auto-linkifies bare URLs; JN preview may not |
| Inline image `![alt](path)` | ✅ | ✅ | ✅ | Relative paths resolve from file's directory |
| HTML comment `<!-- -->` | ✅ | ✅ | ✅ | Hidden in both |
| Horizontal rule `---` | ✅ | ✅ | ✅ | Surround with blank lines |
| Hard line break (2 trailing spaces) | ✅ | ✅ | ✅ | |
| Hard line break (backslash `\`) | ✅ | ✅ | ✅ | GFM extension; both accept |

## GFM extensions

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| Tables (pipe) | ✅ | ✅ | ✅ | Both auto-align via `:---`/`---:`. Pipe inside cell needs `\|` |
| Strikethrough `~~x~~` | ✅ | ✅ | ✅ | |
| Task list `- [ ]` / `- [x]` | ✅ | ✅ | ✅ | GH makes checkboxes clickable in issues/PRs |
| Autolinked URLs (bare) | ✅ | ⚠️ | ✅ | |
| Footnotes `[^1]` | ✅ | ❌ | 🔌 | JN: needs Markdown Navigator or flexmark footnote extension |
| Strikethrough single tilde `~x~` | ❌ | ❌ | ❌ | Use `~~x~~` |

## Diagrams

**Policy: Mermaid only. PlantUML, DOT, D2, drawio, TikZ — forbidden.** The user has Mermaid installed; write fences directly without install instructions. Any non-Mermaid diagram DSL must be refused and converted to Mermaid, or pre-rendered to PNG/SVG and embedded as an image.

| Feature | GH | JN | Notes |
|---|---|---|---|
| ` ```mermaid ` fenced diagram | ✅ | ✅ | User's IDE has Mermaid enabled — no install notes needed. Avoid bleeding-edge diagram types (`block-beta`, `architecture-beta`) for dual-render parity |
| Any non-Mermaid diagram DSL in a code fence (`plantuml`, `dot`, `d2`, `tikz`, `graphviz`) | ❌ | ❌ | **Forbidden.** Refuse and convert |
| Pre-rendered diagram via `![alt](path.svg)` | ✅ | ✅ | Universal fallback for diagrams Mermaid can't express |

## Math

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| Inline `$x$` | ✅ | ✅ | ✅ | No whitespace adjacent to `$`. Escape literal `$` as `\$` |
| Block `$$ x $$` on own paragraph | ✅ | ✅ | ✅ | Blank lines above/below required for both |
| Math in heading | ⚠️ | ⚠️ | ⚠️ | Anchor slug treats `$` oddly — avoid |
| Math in table cell | ⚠️ | ⚠️ | ⚠️ | Renderers may stop at pipe inside formula |
| `\begin{align*} … \end{align*}` | ✅ | ⚠️ | ✅ | GH supports a subset of LaTeX env; JN depends on plugin |
| MathJax macros `\newcommand` | ⚠️ | ⚠️ | ⚠️ | Scope rules differ — define inside the same `$$…$$` block |

## Callouts / Alerts

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| `> [!NOTE]` GFM alert | ✅ | ❌ | 🔌 (≥ 2024 plugin) | JN shows literal `[!NOTE]` text unless very recent plugin. **Major divergence.** |
| `> [!TIP]` / `[!IMPORTANT]` / `[!WARNING]` / `[!CAUTION]` | ✅ | ❌ | 🔌 | Same as above |
| Custom HTML callout (`<table>` or `<div>`) | ⚠️ | ✅ | ✅ | GH strips `style`/`class` — use `<table>` w/ background image, or just `<blockquote>` |
| Admonition `!!! note` (MkDocs syntax) | ❌ | ❌ | ❌ | Not portable |

## Anchors, TOC, navigation

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| Auto-generated heading anchor | ✅ | ✅ | ✅ | Algorithms not identical (Unicode, punctuation, code spans) |
| Explicit anchor `<a id="x"></a>` | ✅ | ✅ | ✅ | Most reliable dual-target anchor |
| Manual TOC | ✅ | ✅ | ✅ | Use explicit anchors for non-ASCII headings |
| Auto-TOC marker `<!-- toc -->` | ❌ | ✅ | ✅ | GH ignores; JN can regenerate via Markdown Navigator |

## HTML

| Element / attr | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| `<details>` / `<summary>` | ✅ | ✅ | ✅ | Best collapsible primitive — fully portable |
| `<sub>`, `<sup>`, `<kbd>`, `<br>` | ✅ | ✅ | ✅ | Safe |
| `<picture>` + `<source media="…">` | ✅ | ✅ | ✅ | Theme-aware images (dark/light) — see github-specific.md |
| `<img>` with `width`/`height` | ✅ | ✅ | ✅ | Allowed attrs |
| `<img>` with `style` | ❌ | ✅ | ✅ | GH strips `style`. Use `width`/`height` attrs instead |
| `<a>` `target="_blank"` | ❌ | ✅ | ✅ | GH strips `target` on most pages; rely on default behavior |
| `<table>` / `<tr>` / `<td>` | ✅ | ✅ | ✅ | Useful when GFM table can't express colspan/rowspan |
| `<div>` / `class` / `id` | ❌ (most) | ✅ | ✅ | GH sanitizer strips `class`/`id` on most elements |
| `<style>`, `<script>`, `<iframe>`, `<form>`, `<input>` | ❌ | ✅ | ✅ | GH strips entirely |
| `align` attribute on block elements | ⚠️ | ✅ | ✅ | GH supports on a small allowlist (`<p align="center">` works) |

## Emoji, mentions, references (GitHub-only sugar)

| Feature | GH | JN | JN+ | Notes |
|---|---|---|---|---|
| Shortcode `:smile:` | ✅ | ❌ | 🔌 | Prefer Unicode emoji 🙂 directly |
| `@username` mention autolink | ✅ | ❌ | ❌ | Only meaningful on GH; safe to write but won't autolink in IDE |
| Issue ref `#123` autolink | ✅ | ❌ | ❌ | Same |
| Commit SHA autolink | ✅ | ❌ | ❌ | Same |
| `user/repo#123` cross-repo ref | ✅ | ❌ | ❌ | Same |

These are GH-only conveniences. They don't break the IDE preview — they just render as plain text. Safe to use; no workaround needed.

## Front matter, metadata

| Feature | GH (repo view) | GH Pages (Jekyll) | JN | Notes |
|---|---|---|---|---|
| YAML `--- … ---` at top of file | ⚠️ shown as HR-bounded section | ✅ parsed & hidden | ✅ hidden | If file is not Jekyll input, **omit front matter** — it renders ugly on repo README |
| TOML `+++ … +++` | ❌ | depends | ❌ | Hugo-specific |

## Special files

| File | GH behavior | JN behavior | Notes |
|---|---|---|---|
| `README.md` at repo / dir root | Auto-rendered on the page | Indexed by IDE | Most-scrutinized file — apply Safe Baseline strictly |
| `.github/*.md` (issue templates, PR templates) | Rendered in issue/PR forms | Plain Markdown in IDE | YAML front matter is **required** here — exception to "no front matter" rule |
| `CHANGELOG.md` | Plain rendering | Plain rendering | Common pattern: H2 = version, H3 = section |
| `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` | GH links to them from repo UI | Plain | |
| Wiki pages | Slightly different renderer (gollum) — supports `[[wikilink]]` syntax | Plain Markdown | Wiki has features the repo renderer does not; not a JN concern |

## Quick "what should I use" cheatsheet

| I want to … | Use this |
|---|---|
| Highlight an important note | GH alert `> [!NOTE]` + accept "plain blockquote with [!NOTE] prefix" in IDE — or use plain `> **Note:** …` if IDE parity matters |
| Show a flowchart | Mermaid `flowchart` inside ```` ```mermaid ```` |
| Show a UML sequence | Mermaid `sequenceDiagram`. **Not** PlantUML — GitHub will not render it |
| Show a class / state / ER / gantt / pie / mindmap diagram | Corresponding Mermaid diagram type — never PlantUML / DOT / D2 / drawio |
| Need a diagram Mermaid genuinely can't express | Pre-render to PNG/SVG and embed via `![]()` — never commit the source DSL in a code block |
| Show math | `$inline$` and `$$block$$` — keep delimiters clean |
| Show a collapsible section | `<details><summary>…</summary> … </details>` |
| Show keyboard keys | `<kbd>Ctrl</kbd>+<kbd>C</kbd>` |
| Show before/after images side-by-side | HTML `<table>` with two `<td>` containing `<img>` |
| Light/dark theme-aware image | `<picture>` + `<source media="(prefers-color-scheme: dark)">` |
| Cross-link inside the document | Explicit `<a id="…"></a>` before the target heading |
| Footnote | `[^1]` — accept that some JN setups will show it literal |
| Center a paragraph | `<p align="center">…</p>` |
