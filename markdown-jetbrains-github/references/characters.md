# Characters, escapes, and encoding

Markdown is a thin layer over plain text. Most "character" problems are not really Markdown problems — they're text-encoding, escape-syntax, or Unicode-vs-ASCII problems that surface when the renderer interprets a character it shouldn't have, or fails to interpret one it should have.

This file is the single source of truth for character-level decisions.

## Table of contents

- [CommonMark backslash escapes](#backslash-escapes)
- [HTML entities](#html-entities)
- [Smart quotes and typographic characters](#smart-quotes)
- [Emoji: Unicode beats shortcodes](#emoji)
- [Unicode in headings (anchor implications)](#unicode-headings)
- [Zero-width characters and BOM](#zero-width)
- [CJK and ASCII spacing](#cjk-spacing)
- [Line endings (LF vs CRLF)](#line-endings)
- [Context-specific escape behavior](#context-escapes)

## <a id="backslash-escapes"></a>CommonMark backslash escapes

CommonMark recognizes backslash escapes for these ASCII punctuation characters, in **any** context that isn't a code block or code span:

```
\ ` * _ { } [ ] ( ) # + - . ! | < > / ' " ~ ^ = $ % & : ; ? @
```

Translation: if you want a literal `*` in prose without italics, write `\*`. If you want a literal `[` without starting a link, write `\[`. The renderers strip the backslash and emit the character.

**Where escaping is necessary (most common):**

| Character | Need to escape when… | Example |
|---|---|---|
| `*` `_` | Used for emphasis at word boundaries | `5 \* 3 = 15` so it doesn't become `5 *3 = 15*` |
| `[` `]` | Looks like the start of a link | `See item \[3\] below` |
| `\|` | Inside a GFM table cell | `\| status: ok \|` (the leading/trailing pipes are the column delimiters) |
| `\\` | When you want a literal backslash | `C:\\Users\\name` |
| `` ` `` | When you want a literal backtick outside a code span | `the \` operator` (rare — usually use code span instead) |
| `<` | Looks like an HTML tag | `2 \< 3` |
| `$` | In a doc with math | `\$5` for a price, vs `$5$` which becomes math |
| `#` | Starts a heading at line start | `\# 5` if you literally want a leading `#` |
| `!` | Precedes `[` and would make an image | `\![not an image](x.png)` |

**Inside code spans and code blocks, backslash is a literal character** — never an escape. Use a code span or a fenced code block whenever you have multiple literal special characters; it's cleaner than backslash soup.

## <a id="html-entities"></a>HTML entities

Both renderers accept named and numeric HTML entities. Useful subset:

| Entity | Renders | When to use |
|---|---|---|
| `&nbsp;` | non-breaking space | Force two words not to wrap apart; pad inside table cells; create blank lines that aren't collapsed |
| `&ensp;` `&emsp;` `&thinsp;` | spaces of different widths | Fine typography in headings or table cells |
| `&zwj;` `&zwnj;` | zero-width joiner / non-joiner | Composite emoji, CJK / Indic correctness |
| `&shy;` | soft hyphen | Optional line-break hint in long words (rarely needed in Markdown) |
| `&amp;` | `&` | When the literal `&` could be mistaken for an entity. Plain `&` works in most places but `&amp;` is safer |
| `&lt;` `&gt;` | `<` `>` | When `<` could start an HTML tag and you don't want backslash escapes (e.g. inside HTML blocks) |
| `&quot;` `&apos;` | `"` `'` | Inside HTML attribute values |
| `&hellip;` | `…` | Prefer the literal `…` character |
| `&mdash;` `&ndash;` | `—` `–` | Prefer the literal characters |
| `&copy;` `&reg;` `&trade;` | `© ® ™` | Prefer literals |
| `&times;` `&divide;` | `× ÷` | Decorative only (e.g. a heading "Cross × Cross"). **Never** as math — mathematical expressions must use LaTeX (`$\times$`, `$\div$`). See `math.md` |
| `&larr;` `&rarr;` `&uarr;` `&darr;` | `← → ↑ ↓` | Prefer literals |

**General principle: prefer the literal Unicode character.** Use entities only when (1) the literal would be ambiguous in context (`&` near another `&entity;`), (2) you're authoring inside an HTML block where the entity reads more clearly, or (3) the character is hard to type and the entity is mnemonic.

`&nbsp;` is the most common reason to reach for entities — it's the only practical way to force a non-breaking space because the literal Unicode `U+00A0` is invisible in the source.

**Hard exception for math:** the "prefer the literal Unicode character" principle does **not** apply to mathematical content. Mathematical expressions, including any Greek letter, operator, set-theory symbol, integral, sum, fraction, or comparison that appears as part of a formula, must be written in LaTeX inside `$…$` or `$$…$$`. Never use raw Unicode (α, Σ, ∫, ≤, →, ½, ∞) as a substitute for LaTeX commands in a mathematical expression. See `math.md` for the full policy and exceptions (proper nouns, non-formula diagram labels).

### Entity edge case: GFM tables

Inside GFM pipe tables, `&nbsp;` is the only way to produce a visually empty cell that still occupies space:

```markdown
| col1 | col2 |
|---|---|
| value | &nbsp; |
```

A truly empty cell (`| value |  |`) renders without padding on GitHub and may collapse the column visually.

## <a id="smart-quotes"></a>Smart quotes and typographic characters

Both GitHub and JetBrains render Markdown **without** automatic smart-quote substitution. `"hello"` stays as straight quotes. `--` stays as two hyphens (not an en-dash). `...` stays as three dots (not an ellipsis).

If you want typographic characters, **type them directly**:

| Want | Type | Don't rely on |
|---|---|---|
| Em dash | `—` (U+2014) | `---` auto-conversion |
| En dash | `–` (U+2013) | `--` auto-conversion |
| Ellipsis | `…` (U+2026) | `...` auto-conversion |
| Curly double quote | `"` `"` (U+201C / U+201D) | `"..."` auto-conversion |
| Curly single quote / apostrophe | `'` `'` (U+2018 / U+2019) | `'...'` auto-conversion |
| Middle dot | `·` (U+00B7) | none |
| Bullet | `•` (U+2022) | none |

For technical writing, straight quotes are usually correct (especially around code-like strings — curly quotes break copy-paste). Reserve typographic characters for narrative prose.

**Watch out:** input methods (macOS "Smart Quotes" auto-correct, some Chinese IMEs) silently insert curly quotes into source files. If you're documenting code and see `console.log("hello")` rendering as `console.log("hello")` with curly quotes, an IME or editor setting did it. Disable that for `.md` files.

## <a id="emoji"></a>Emoji: Unicode beats shortcodes

GitHub converts shortcodes like `:smile:` → 😄. JetBrains' bundled plugin does not — it renders the literal text. To stay dual-target, **always write the Unicode emoji character directly**.

```markdown
✅ Good — works in GitHub, JetBrains, and any other CommonMark renderer
⚠️ Be careful — same
❌ Avoid this pattern — same

✗ Bad: :white_check_mark: — GitHub renders ✅, JetBrains shows literal `:white_check_mark:`
```

How to type Unicode emoji:
- macOS: Control + Command + Space
- Linux: Ctrl + Shift + U then code point
- Windows: Win + .  (period)
- Any platform: copy from https://emojipedia.org or paste from another file

**Composite emoji** (skin-tone modifiers, ZWJ sequences for "woman scientist", flags) are a sequence of code points joined by U+200D (zero-width joiner). They render correctly in modern browsers and modern IDE preview panes; older JetBrains versions may show the components separately.

## <a id="unicode-headings"></a>Unicode in headings (anchor implications)

Headings with non-ASCII characters (CJK, accented Latin, emoji, mathematical symbols) generate slugs that **differ between GitHub and JetBrains**:

- GitHub: keeps non-ASCII characters in the slug, URL-encodes them in the rendered `id`.
- JetBrains: behavior varies by plugin version — some strip non-ASCII, some keep them, some normalize differently.

For headings like `## 数据流 / Data Flow` or `## 性能基准 (P99)`, the auto-generated anchor will not be portable. **Use explicit anchors**:

```markdown
<a id="data-flow"></a>
## 数据流 / Data Flow

…later in the doc…

See the [data flow](#data-flow) section.
```

The `<a id="…">` form gives you a stable, ASCII, slug-controlled anchor that works in every renderer. Use it whenever the heading text contains anything outside `[a-z0-9-]`.

## <a id="zero-width"></a>Zero-width characters and BOM

These are invisible characters that can appear in source files via copy-paste from web pages, PDFs, or Notion/Confluence exports:

| Character | Code point | What it does | What to do |
|---|---|---|---|
| Byte Order Mark | U+FEFF | Marks UTF-16 endianness; at start of UTF-8 file it's noise | Strip from file start; many editors hide it |
| Zero-width space | U+200B | Breaks word boundary without visible space | Strip unless intentional |
| Zero-width non-joiner | U+200C | Prevents ligature | Strip unless typographic intent |
| Zero-width joiner | U+200D | Forms composite emoji and Indic conjuncts | Keep — usually intentional |
| Word joiner | U+2060 | Forbids line break | Strip unless intentional |
| Left-to-right/right-to-left mark | U+200E / U+200F | Bidi control | Strip unless mixed-direction text |
| Non-breaking space | U+00A0 | Visible as space, but doesn't break or collapse | Often intentional (see entities) |

GitHub renders all of these as invisible characters. JetBrains' preview behaves the same. The damage is to the **source** — code reviews show weird diffs, grep doesn't find words, and copying examples breaks because the invisible character travels with the paste.

**Detection** (in JetBrains): Settings → Editor → General → Appearance → "Show whitespaces" + "Show special characters". Or run a one-liner:

```bash
LC_ALL=C grep -nP '[\x{200B}-\x{200F}\x{2060}\x{FEFF}]' README.md
```

## <a id="cjk-spacing"></a>CJK and ASCII spacing

When CJK characters (Chinese / Japanese / Korean) sit next to ASCII letters or digits, **don't insert spaces unless the typography style of the project requires it**. CommonMark/GFM and the JetBrains plugin both treat the boundary correctly without explicit spaces.

Both styles are valid:

```markdown
✅ Style A — no space at boundaries (typical in mainland-China tech docs)
   今天发布了 v2.3 版本,新增 3 个 API。

✅ Style B — space at boundaries (typical in Taiwan/Anthropic-style docs)
   今天发布了 v2.3 版本，新增 3 个 API。
```

(Note: style B above uses full-width punctuation `，` `。`, not half-width `,` `.`. Mixing half-width punctuation in CJK text is the more common style issue.)

A few real failure modes:

1. **Linkified CJK with no boundary**: `查看 [文档](docs/)。` — works. `查看[文档](docs/)。` — also works (CommonMark allows attached link text). Both renderers parse correctly.
2. **Emphasis inside CJK**: `这是**重点**` — works on GitHub, may fail on JetBrains older plugins because `**` requires whitespace-or-punctuation boundaries in strict CommonMark. **Workaround**: `这是 **重点**` with spaces, or `这是<b>重点</b>` with HTML.
3. **Full-width vs half-width digits**: `Ｐ９９` (full-width) is different from `P99` (half-width). Anchor slugs, sorting, and search treat them as distinct. Prefer half-width for technical terms.

## <a id="line-endings"></a>Line endings (LF vs CRLF)

Markdown spec is line-ending-neutral, but two operational gotchas:

1. **Mixed line endings within a file** (some lines LF, some CRLF) can confuse JetBrains' formatter and some Git diff tools, even though both renderers display fine. Settings → Editor → Code Style → Line separator → `\n` for `.md` files. Or rely on `.gitattributes`:
   ```
   *.md text eol=lf
   ```
2. **Trailing whitespace doubles as a hard-break syntax** (`text<space><space>` at end of line forces a `<br>`). Many editors strip trailing whitespace on save, which silently removes intentional hard breaks. If hard breaks matter, prefer the backslash form `text\` (also GFM-supported), which survives whitespace stripping.

## <a id="context-escapes"></a>Context-specific escape behavior

The same character means different things in different contexts. Quick reference:

| Context | Special chars | Notes |
|---|---|---|
| Prose | `\ * _ [ ] ( ) # + - . ! | < > $` | Use backslash escapes |
| Inline code `` ` `` | None — everything is literal | Use double backticks `` `` ` `` `` to embed a backtick |
| Fenced code block | None — everything is literal | Language id after opening fence is parsed |
| Link `[text](url)` | `( )` inside the URL: escape as `\(` `\)` or URL-encode | `[](https://a.com/foo\(bar\))` |
| Link `[text](url "title")` | `"` inside title: escape as `\"` | Or use single quotes for the title delimiter |
| GFM table cell | `\|` for literal pipe; line breaks via `<br>` | No code blocks inside cells |
| `$ math $` | LaTeX rules apply; `\` is LaTeX, not Markdown | Use `$$ ... $$` to write multi-line math |
| HTML block | HTML rules; Markdown is **not** re-processed inside most HTML blocks | Add a blank line and indent to make Markdown render inside `<div>` |

The general principle: **the innermost-nested syntax wins.** Inside a code span, backslash is a literal. Inside math, backslash is LaTeX. Inside a link's URL, backslash escapes parentheses. The outer Markdown's escape rules don't apply.
