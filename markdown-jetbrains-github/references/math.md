# Math (LaTeX) in dual-target Markdown

Both GitHub and JetBrains render `$inline$` and `$$block$$` math, but with different engines and different edge cases. This file is the single source of truth for math in this skill.

- GitHub: MathJax v3, server-rendered to inline SVG/HTML.
- JetBrains: KaTeX or MathJax depending on plugin version, rendered in the JCEF preview pane.

The two engines agree on the **common LaTeX subset** the average user writes. They disagree on TeX environments, macros, and a few delimiter edge cases. Stay inside the common subset whenever possible.

## <a id="latex-only"></a>The LaTeX-only policy

**Every mathematical expression, formula, and standalone mathematical character used in a mathematical sense MUST be written in LaTeX inside `$…$` or `$$…$$`.** Raw Unicode (α, Σ, ∫, ≤, →, ½, xᵢ) is never an acceptable substitute for LaTeX commands inside math.

This applies even to seemingly trivial cases — a single `α` denoting a learning rate is math, write `$\alpha$`. A `≤` in a constraint is math, write `$\leq$`. An `∞` in a limit is math, write `$\infty$`.

Examples (each row is one fix that must be applied automatically — don't ask the user):

| Wrong | Right |
|---|---|
| `the rate α controls learning` | `the rate $\alpha$ controls learning` |
| `loss = Σ (yᵢ − ŷᵢ)²` | `$\text{loss} = \sum_i (y_i - \hat{y}_i)^2$` |
| `n → ∞` (as a limit claim) | `$n \to \infty$` |
| `O(n log n) complexity` | `$O(n \log n)$ complexity` |
| `∀ x ∈ ℝ, f(x) ≥ 0` | `$\forall x \in \mathbb{R}, f(x) \geq 0$` |
| `½ + ¼ = ¾` | `$\tfrac{1}{2} + \tfrac{1}{4} = \tfrac{3}{4}$` |
| `complexity ≤ O(n²)` | `complexity $\leq O(n^2)$` |
| `x ∈ [0, 1]` | `$x \in [0, 1]$` |

**The only exceptions** — narrow, and require deliberate judgment:

- **Proper nouns / identifiers** that *resemble* math but are names: a product called "Alpha 2", an algorithm named "Σ-protocol", a file `n_β.csv`, a brand or version. Not math.
- **Mermaid diagram labels**: Mermaid doesn't run MathJax, so LaTeX inside a label won't render. If a label needs a single Greek letter, use raw Unicode in the label — but **never** put a multi-symbol formula in a label. Move formulas to a `$$…$$` block adjacent to the diagram.
- **Pure narrative arrows** describing a pipeline (`Client → Parser → Engine → Output`) where the `→` is a process step, not a mathematical limit/mapping. As soon as the arrow has mathematical meaning (`n → ∞`, `f: A → B`), it's math — use LaTeX.

When in doubt, treat it as math.

## Table of contents

- [The LaTeX-only policy](#latex-only)
- [Delimiters and the four iron rules](#delimiters)
- [What's safe to write](#safe-subset)
- [Supported environments](#environments)
- [Macros and `\newcommand`](#macros)
- [Forbidden positions: tables and headings](#forbidden-positions)
- [Verifying math renders on both sides](#verifying)

## <a id="delimiters"></a>Delimiters and the four iron rules

### Inline math: `$...$`

```markdown
The energy is $E = mc^2$ and the period is $T = 2\pi / \omega$.
```

### Block math: `$$...$$`

```markdown
The Gaussian integral is

$$
\int_{-\infty}^{\infty} e^{-x^2} \, dx = \sqrt{\pi}
$$

and it underlies the normal distribution.
```

### The four iron rules

Break any of these and one of the two renderers will silently fail.

1. **No whitespace adjacent to inline `$` delimiters.** `$ x $` ❌ — GitHub's heuristic disqualifies it (designed to avoid matching dollar prices `$5 to $10`). Use `$x$` ✅.

2. **Block math is its own paragraph.** Blank line above, blank line below, no text on the same lines as the `$$`. Inline-style `text $$x$$ more text` is unreliable on both renderers — pick inline (`$x$`) or block, never block-on-a-line.

3. **Escape literal `$`.** If the document mentions prices alongside math, write `\$5` for the price. Otherwise GitHub will eat the `$` while looking for a closing delimiter.

4. **Pipe `|` inside math conflicts with tables.** Escape as `\|` inside formulas that live in table cells, or move the math out of the table. See [forbidden positions](#forbidden-positions).

## <a id="safe-subset"></a>What's safe to write

The common LaTeX subset both renderers handle reliably:

- All ASCII math: `+ - * / = < > ( ) [ ]`
- Greek letters: `\alpha`, `\beta`, ..., `\Omega`
- Sub/superscript: `x_i`, `x^2`, `x_i^2`, `x^{(n+1)}`
- Fractions: `\frac{a}{b}`, `\tfrac{a}{b}`, `\dfrac{a}{b}`
- Roots: `\sqrt{x}`, `\sqrt[n]{x}`
- Sums/integrals/products: `\sum`, `\int`, `\prod`, with `_` / `^` for limits
- Limits and operators: `\lim`, `\inf`, `\sup`, `\log`, `\ln`, `\exp`, `\sin`, `\cos`
- Vectors and accents: `\vec{x}`, `\hat{x}`, `\bar{x}`, `\dot{x}`
- Spacing: `\,` (thin), `\;` (medium), `\quad`, `\qquad`
- Brackets that auto-size: `\left( ... \right)`, `\left\{ ... \right\}`
- Common symbols: `\to`, `\Rightarrow`, `\leq`, `\geq`, `\neq`, `\approx`, `\equiv`, `\in`, `\subset`, `\cup`, `\cap`, `\emptyset`, `\infty`, `\partial`, `\nabla`

If you're writing physics/ML/engineering formulas, the above covers 95 % of cases without environment-level constructs.

## <a id="environments"></a>Supported environments

Safe in both renderers: `aligned`, `cases`, `array`, `matrix` / `pmatrix` / `bmatrix` / `vmatrix`. Use `aligned` *inside* a `$$…$$` block for multi-line equations — it's the most portable pattern:

```markdown
$$
\begin{aligned}
  a &= b + c \\
    &= d
\end{aligned}
$$
```

Not supported anywhere (don't write these): `theorem` / `proof` / `lemma` / `definition` (use Markdown heading + body instead), `tikzpicture` (pre-render to image), `align*` standalone (replace with `aligned` inside `$$…$$`).

## <a id="macros"></a>Macros and `\newcommand`

**`\newcommand` is scoped to a single math block on GitHub** — it does not persist across blocks. JetBrains behavior depends on plugin version, but assume the same scoping.

Implications:

1. To reuse a macro, redefine it at the top of every `$$ … $$` block that needs it.
2. For long documents, the cleanest pattern is a one-time **hidden setup block** containing the definitions — but it still only scopes to that block, so you have to redefine inside each math block anyway.
3. `\renewcommand` and `\def` follow the same scoping.
4. `\require{}` for MathJax extensions (e.g. `\require{cancel}`) **works on GitHub** but may not in JetBrains; avoid for dual-target.

If you find yourself defining the same macro repeatedly, that's a sign the math is dense enough that a separate doc or a PDF artifact is more appropriate than inline Markdown math.

## <a id="forbidden-positions"></a>Forbidden positions: tables and headings

### Math inside table cells

Pipe-table cells use `|` as the column separator. LaTeX uses `|` for absolute value and set-builder notation (`\{x | x > 0\}`), so naïve math inside a cell **breaks the table layout**:

```markdown
| f(x) | derivative |
|---|---|
| $|x|$ | $\text{sign}(x)$ |
```

Both renderers parse `$|x|$` as `$`, then a column separator, then `x`, then another column separator — three cells, not one.

Three workarounds, in order of preference:

1. **Move math out of the cell.** Use a numbered caption and a math block below:

   ```markdown
   | f(x) | derivative |
   |---|---|
   | (1) | (1') |

   1. $|x|$ — $1' = \text{sign}(x)$
   ```

2. **Escape pipes in the formula** as `\|`:

   ```markdown
   | $\lvert x \rvert$ | $\text{sign}(x)$ |
   ```

   The `\lvert ... \rvert` form avoids the pipe entirely, which is the most portable fix.

3. **Use an HTML `<table>`** instead of a GFM pipe table — full freedom inside cells, at the cost of much more markup.

### Math inside headings

Both renderers produce surprising anchor slugs when `$` appears in heading text. Avoid:

```markdown
## The integral of $f(x)$
```

GitHub slug: `the-integral-of-fx` (drops `$` and parens, joins). JetBrains slug: similar but not identical for the inner text. If anyone links to `#the-integral-of-fx`, it may or may not work depending on the renderer.

Fixes, in order of preference:

1. **No math in headings.** Restate the formula in prose right after:
   ```markdown
   ## The integral of f

   For $f(x) = x^2$ the integral is …
   ```
2. **Explicit anchor**:
   ```markdown
   <a id="integral-of-f"></a>
   ## The integral of $f(x)$
   ```
   Then link to `#integral-of-f`. Anchor is decoupled from the slug algorithm.

## <a id="verifying"></a>When renders differ

If the GitHub and JetBrains previews diverge, the cause is almost always one of: an unsupported environment, a macro that didn't carry across blocks, whitespace inside `$…$`, or math placed inside a table cell or a heading. Fix at the source — never maintain two copies of the math.
