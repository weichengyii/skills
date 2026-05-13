# JetBrains IDE Markdown — specifics

JetBrains IDEs (IntelliJ IDEA, PyCharm, GoLand, WebStorm, Rider, RustRover, CLion, PhpStorm, RubyMine, AppCode, Writerside, DataGrip) all ship with the same **bundled Markdown plugin**, which uses two parsing backends:

- **intellij-markdown** — Kotlin-Multiplatform parser, default `CommonMarkFlavour` plus a GFM-extension flavor. Used for highlighting, completion, structure, navigation, and the basic preview.
- **flexmark-java** — used when richer rendering or extension support is needed (e.g. as the engine behind the Markdown Navigator commercial plugin, which historically replaced pegdown).

Read this when working on the IDE side of a dual-target Markdown file.

## What is bundled (works out of the box)

- CommonMark + GFM core: headings, lists, fenced code, tables, task lists, strikethrough, autolinks (limited)
- Live preview pane (JCEF browser embedded), split editor/preview
- Language injection inside fenced code blocks — `# ```python` makes the IDE treat that block as real Python with completion, inspections, and refactoring
- Path completion for image and link targets relative to the file
- Heading navigation in the structure tool window
- Reformat (configurable in Settings → Editor → Code Style → Markdown)
- Soft-wrap by default in `.md` files (toggle in Settings → Editor → General)
- Custom CSS for the preview (Settings → Languages & Frameworks → Markdown → Custom CSS)
- Export to HTML; PDF/DOCX via Pandoc if installed

## Markdown Extensions (toggleable, still bundled)

Settings → Languages & Frameworks → Markdown → Markdown Extensions:

- **Mermaid** — required for ` ```mermaid ` fences to render. **The user has this enabled already** (see SKILL.md Environment assumptions). Do not instruct the user to install or enable anything related to Mermaid.

> This skill **forbids** PlantUML, regardless of any PlantUML extension or plugin available in JetBrains. PlantUML does not render on GitHub. See SKILL.md "Environment assumptions".

## Optional plugins (Marketplace)

When the bundled support is not enough:

| Plugin | What it adds | When to recommend |
|---|---|---|
| **Mermaid** (20146-mermaid) | First-class `.mmd`/`.mermaid` file type, syntax highlighting, completion, live preview, snippets (`mflow`, `mseq`, …) | User authors many Mermaid diagrams |
| **Mermaid Studio** (29870-mermaid-studio) | Like above, with richer editing UI | Alternative to the above |
| **Markdown Navigator** (vsch/idea-multimarkdown, commercial Enhanced edition) | Pegdown-replacement parser (flexmark), footnotes, definition lists, abbreviations, custom TOC, attribute syntax, table editing, paste image as link | Heavy Markdown authoring (docs, books) |
| **Writerside** (also a standalone JetBrains IDE) | Topic-based authoring, semantic markup, build to static site | Product documentation; out of scope for typical README work |

> Plugins for PlantUML, Diagrams.net, Graphviz/DOT, and similar non-Mermaid diagram backends are deliberately omitted. Even though they work inside JetBrains, the output does not render on GitHub, which violates the dual-target premise of this skill. Stick to Mermaid.

## Language injection — the IDE's superpower

A fenced code block with a recognized language id is treated by the IDE as real code:

````markdown
```rust
fn main() {
    println!("hello");
}
```
````

Inside that fence, the user gets:
- Full syntax highlighting (not just colored tokens — semantic highlighting)
- Code completion
- Inspections (warnings, type errors, lints)
- Navigation (Ctrl+Click on identifiers — though resolution scope is limited to the block)
- Reformat with the language's code style

GitHub also highlights, but does not provide intelligence. So a code sample that looks fine on GitHub may still surface real errors in the IDE — useful for catching issues in documentation snippets.

Common language IDs the IDE recognizes for injection: `kotlin`, `java`, `python`, `js`/`javascript`, `ts`/`typescript`, `go`, `rust`, `c`, `cpp`, `cs`/`csharp`, `php`, `ruby`, `swift`, `sql`, `shell`/`bash`/`sh`, `yaml`, `json`, `toml`, `xml`, `html`, `css`, `scss`, `dockerfile`, `groovy`, `gradle`, `properties`, `regex`. Plus `mermaid` for diagram fences (the only diagram backend this skill uses).

If a language is not recognized, the block still renders as a code block in both IDE preview and GitHub — language id just doesn't trigger highlighting.

## What does NOT render in the bundled plugin

| Feature | Status | Workaround |
|---|---|---|
| GFM alerts `> [!NOTE]` | Renders as plain blockquote with `[!NOTE]` literal | Wait for newer plugin version, or use `> **Note** —` plain-prose style |
| Footnotes `[^1]` | Renders literally | Markdown Navigator plugin, or accept that footnotes only render on GH |
| Definition lists | No | Markdown Navigator |
| Abbreviations | No | Markdown Navigator |
| `:emoji_shortcode:` | No | Use Unicode emoji directly |
| `@user`, `#123`, SHA autolinks | No | These are GH-only by design — safe to leave plain |
| MkDocs `!!! note` admonitions | No | Use GFM alerts (and accept the IDE downgrade) |
| Wikilinks `[[link]]` | No | Use `[text](path)` |
| Strikethrough single tilde `~x~` | No | Use `~~x~~` |

## File types associated by default

- `.md` ✅
- `.markdown` ✅
- `.mdown` ✅
- `.mkd`, `.mkdn` ✅
- `.mdx` — recognized as Markdown but JSX is not parsed (it stays as text inside the preview). For real MDX support, use the MDX plugin.

## Reformat behavior

`Code → Reformat File` on a `.md` will normalize:
- Trailing whitespace
- Heading style (per Code Style settings — ATX or Setext)
- List bullet character (per Code Style)
- Wrap at right margin (off by default — turning this on can rewrap prose paragraphs, which surprises users; check before applying)
- Table column alignment (re-aligns `|` separators visually)

When working on someone else's Markdown file, **do not run Reformat unprompted** — it can produce a noisy diff.

## How to verify behavior in the IDE

When the user reports "X doesn't render in my IDE preview":

1. Check the IDE version and the Markdown plugin version (Settings → Plugins → Installed).
2. Check Markdown Extensions (Settings → Languages & Frameworks → Markdown → Markdown Extensions) — is Mermaid enabled?
3. Check if any third-party Markdown plugin is installed and intercepting rendering.
4. Try the View → Appearance → Preview Layout toggles — sometimes the preview is hidden.
5. For Mermaid specifically: the first render after enabling downloads a JS bundle; offline sessions may fail silently.

## Custom CSS for preview

Settings → Languages & Frameworks → Markdown → Custom CSS. Two modes:

- Path to a `.css` file
- Inline CSS in the settings box

Common ask: make the preview match `github-markdown.css` for cross-renderer parity. The user can drop a copy of `github-markdown.css` (MIT-licensed) into the project and point Custom CSS at it. After that, what they see in the IDE preview is very close to what GitHub renders, modulo the unsupported features above.
