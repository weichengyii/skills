---
name: research-first-algorithm
description: MUST invoke this skill FIRST, BEFORE writing any code, whenever the user asks to implement, add, write, port, rewrite, fix, or replace any non-trivial algorithm. EXAMPLE TRIGGER QUERIES (both English and Chinese count): "implement Moorer reverb", "帮我在 effects/reverb 里实现 X reverb", "add a pYIN pitch detector", "在 analysis 里加个 pitch detection", "写一个 Karplus-Strong 古筝", "port WORLD to Rust", "给 filter 加个 Linkwitz-Riley crossover", "implement RNNoise", "write Butterworth 8th-order LP with matched-z", "add CREPE", "帮我写个 shimmer reverb", "add lookahead limiter", "implement ISTFT with COLA reconstruction", "port the <library>'s <algorithm> to <language>". TRIGGER CATEGORIES: (1) algorithms named after a person/paper/institution (Dattorro plate, Jot FDN, Schroeder/Moorer reverb, YIN, pYIN, WORLD, PSOLA, Karplus-Strong, RBJ cookbook biquads, Butterworth/Chebyshev/Linkwitz-Riley design, FFT variants, NLMS, RLS, RNNoise, CREPE, SPICE, pesto, McAulay-Quatieri, Roex auditory filter, MFCC, mel-spectrogram, phase vocoder, Viterbi, Kalman, Steiglitz-McBride, Burg, Levinson-Durbin, Goertzel, etc.); (2) anything originating from a specific paper or standard; (3) any port from another language/framework (JUCE/Faust/librosa/scipy/MATLAB/C reference/pytorch/torchaudio); (4) any multi-stage/composite DSP or ML algorithm; (5) your own uncertainty — if you catch yourself thinking "I think it works like...", that is the trigger. DO NOT treat these requests as ordinary coding tasks you can write from memory — that is exactly the failure mode this skill prevents. Memory-based implementations of named algorithms routinely ship with silently-wrong coefficients, missing stages, or wrong sign conventions, and the skill prevents that by enforcing: primary-paper lookup + ≥2 independent reference implementations + survey/textbook for context; a comprehensive 300–600 line research brief (historical context, derivation, equations, topology, ref-impl audit, complexity, domain quality criteria, parameter sensitivity, failure-mode gallery, validation) saved to docs/algorithms/<name>.md; explicit user approval at a hard gate BEFORE writing code; and a blind sub-agent independent review AFTER implementation. SKIP ONLY FOR: trivial routines (gain stage, linear mix, unit conversion, simple linear interpolation), pure refactors preserving behavior (renames, inline-annotation changes, moving code between files), obvious one-line bug fixes (typo in variable name, off-by-one in loop, wrong comparison operator), pure explanation of existing code without modification, README/doc edits, and tooling/Cargo/build config changes.
---

# Research-First Algorithm Implementation

## Who this skill is for

Algorithm researchers and DSP engineers. The user has explicitly chosen to pay latency for correctness and understanding — honor that choice. Do NOT optimize for "one-shot it fast". Optimize for: the user finishing this session with deeper knowledge of the algorithm, implementations that match published references, and briefs that become permanent project memory.

Brevity is not a virtue here. A 600-line brief that captures the algorithm's historical lineage, derivation, numerical behavior, and failure modes is more valuable than a 150-line brief that lists equations. If you feel the impulse to "keep it concise", suppress it — ask instead "what else should a researcher know before touching this code?"

## Why this skill exists

LLMs carry strong associative memory of classical algorithms. The memory is just good enough to produce code that compiles, runs, and sounds/looks/measures plausible — but silently differs from the published algorithm by a sign, a coefficient, a delay sample, a missing stage, or a mistaken topology. These errors survive casual testing. They ship. They're expensive to find later.

Worse: **shallow implementations produce shallow debugging.** When something doesn't sound right, the engineer without a deep mental model can only tweak knobs. The engineer with a deep mental model knows which knob is actually the problem, or — more importantly — knows the code itself is the problem and can point to the specific equation that was implemented wrong.

Human researchers don't work from memory on this class of problem. They:
1. Read the paper (in full, not the abstract) and the papers the paper cites
2. Read at least two reference implementations from different authors — critically, noting where they differ and from each other and from the paper
3. Read a survey or review article that places the algorithm in context — what came before, what came after, what's the current state of the art
4. Derive the key equations themselves, or verify the derivation step-by-step
5. Understand the algorithm's *failure modes* — what wrong looks like, what causes it, how to detect it
6. Only then do they write code

This skill turns that discipline into a mandatory workflow. Briefs produced by this skill are not summaries — they are study notes, deep enough that a future engineer (or future-you in six months) can re-derive intent from them.

## When to trigger

Trigger **before** writing implementation code if any of the following hold:

- The algorithm has a name tied to a person, paper, institution, or technique
- The algorithm originates in a specific paper, book, or standard
- Coefficients, weights, filter shapes, delay lengths, window functions, mixing matrices, or other "magic numbers" would come from memory
- Sign conventions, sample-rate-dependent constants, or delay compensation are involved
- You're porting code from another language, framework, or paper pseudocode
- The implementation composes two or more known algorithms
- You are not 100% certain about any of: parameter meanings, structural topology, underlying principles, numerical pitfalls, validation methodology, or the algorithm's place in the field

The uncertainty test is load-bearing: **"I think this works like X"** is a trigger. Memory is not a source.

## Do NOT trigger for

- Trivial routines (gain stage, linear mix, unit conversion, simple interpolation)
- Reading/explaining existing code without modifying it
- Bug fixes where the algorithm is in place and the fault is obvious (off-by-one in a loop, wrong variable name, typo in a symbol)
- Performance optimization of already-correct code (same math, different layout)
- Refactors that preserve behavior
- Pure tooling / configuration / CI / documentation work

When in doubt, trigger. The cost of an unnecessary brief is a wasted afternoon. The cost of a silently-wrong algorithm shipping is weeks of confused debugging.

## The workflow

Five phases. Do not skip or reorder. The ordering is the whole point.

---

### Phase 1 — Research (go deep, not just wide)

**Research for understanding, not just for citations.** The brief is an artifact; the understanding is the product. A brief you could produce without actually comprehending the algorithm is a failed brief.

Gather material from **at least three independent kinds of sources**:

#### Source kind 1 — Primary source (the paper / standard)

The original publication: paper, textbook chapter, or standard document. Record authors, title, venue, year, DOI, and a stable link.

Read the paper **in full**, not just the abstract or conclusion:
- Understand the problem the paper is solving and why the prior state of the art was insufficient
- Trace at least the central derivation(s) — ideally, derive it again yourself in the brief and confirm you get the paper's answer
- Note equation numbers; preserve the paper's notation when quoting
- Pay attention to the paper's *own* discussion of limitations, failure modes, edge cases, and parameter choices

If the primary source is paywalled, locate an author's preprint, accepted manuscript, or a citation-equivalent textbook treatment. If the PDF is a scanned image (common for pre-1990 papers), flag this explicitly and cross-check numerical values against at least two independent secondary sources.

#### Source kind 2 — Reference implementations (at least two, preferably three)

Minimum **two reference implementations from different authors**. Priority order:
- Original author's reference code (published with the paper or a companion artifact)
- Widely-used libraries with DSP/ML track record (JUCE, Faust, librosa, scipy.signal, MATLAB/Octave toolboxes, CSound opcodes, essentia, aubio, torchaudio, TensorFlow audio)
- Well-maintained GitHub repositories (check stars, issue quality, commit recency, test coverage — as *signal*, not certification)

**Read the code. Do not just collect links.** For each reference implementation, record:
- What the code teaches you that the paper doesn't (initial conditions, denormal guards, edge cases, parameter defaults that work in practice)
- Where it agrees with the paper
- Where it deviates — and state your assessment of who is right, with reasoning
- Overall trust level for this source, on what dimensions

If reference implementations disagree with each other, that disagreement is a **question to answer, not a vote to tally**. Decide which source you trust for which fact and explain why. Contradictions unresolved in the brief become bugs in the code.

#### Source kind 3 — Survey, review, textbook, or modern successor

A survey or review article, a textbook chapter, or a paper on a successor/competitor algorithm — something that places the target algorithm **in context**. This is what distinguishes an informed implementation from a historical one. Questions it should help answer:

- Why was this algorithm proposed? What came before?
- Has it been superseded? By what? When?
- What are its known failure regimes that later work has identified?
- In what range of applications is it still the recommended choice?
- Are there improvements / extensions that have become standard practice beyond the original paper?

Examples: Välimäki et al. "Fifty Years of Artificial Reverberation" (2012) for reverbs; de Cheveigné's successor work on pitch detection; DAFX textbook chapters; JASA tutorial reviews; recent CREPE / SPICE / pesto papers as successors to YIN / pYIN.

#### Depth discipline

While researching, practice these habits:

- **Question the abstract.** Papers' abstracts always oversell. The limitations are usually buried in section 5-6.
- **Derive, don't memorize.** If the paper says `g = 10^(-3·d/(RT60·SR))`, work out yourself why that gives a 60 dB decay in RT60 seconds. Put that derivation in the brief.
- **Find the "extra" from the code.** Reference implementations contain wisdom that's not in any paper — start-up conditions, denormal guards, numerical reformulations. Capture it.
- **Pay for access when needed.** If a key paper is paywalled and you have access via academic credentials, use them. If not, document what's inaccessible and work around it — but flag the uncertainty in the brief.
- **For audio DSP specifically:** listen to reference implementations' output when possible (audio examples, IRs, test signals). The ear catches classes of bug that numerical tests miss.

---

### Phase 2 — Write the brief

Save to `docs/algorithms/<snake_case_algorithm_name>.md` in the project root. Create the directory if it doesn't exist. If there's no git repo or project root, fall back to `~/.claude/research-briefs/<name>.md`.

**Target length: 300–600 lines.** This is a working study note, not a dissertation — but don't cut sections that capture real understanding to hit a word count. If a section is thin because the algorithm really is simple on that axis, say so in the section and move on. If a section feels thin because you rushed, go back and research more.

Use this structure:

```markdown
# <Algorithm Name>

## Primary source
- Authors, "Title", Venue, Year. DOI or stable URL.
- Additional corroborating sources (textbook chapters, standards).

## Historical context & place in the field
Where does this algorithm sit in the lineage?
- What came immediately before and what problem did this paper solve that predecessors couldn't?
- What successors exist? Have any superseded this for new work?
- What applications still use this as the recommended choice, and why?
- Reference one survey/review/successor paper by citation.

A reader should finish this section knowing whether they're looking at state-of-the-art, a widely-used workhorse, or a historical curiosity.

## What the algorithm does
One paragraph. The problem it solves and the key insight, stated plainly. A reader who has never heard of the algorithm should understand what it's *for* after this paragraph.

## Intuition / why it works
**WHY** it works, not just what. A paragraph or two that conveys the key insight in plain language, ideally with an analogy or a limiting case. If the algorithm exploits a statistical property, a physical model, or a mathematical identity, name it.

This section exists because code written without intuition is code you can't debug when it goes wrong.

## Derivation of key equations
Not just quoting equations — deriving them. At minimum, sketch the path from the underlying insight to the key equation(s). Show why the constants come out the way they do.

Preserve paper notation and equation numbers:

    y[n] = a * x[n] + b * y[n-1]    (eq. 3)

Define every symbol. If any symbol's meaning required interpretation on your part, flag it.

If there are alternative formulations in different references (e.g. one-pole LPF as `(1-d)·x + d·y_prev` vs `(1-d)/(1-d·z^-1)` transfer function vs two-coefficient lfilter form), list them and prove (or reason through) their equivalence.

## Topology / block diagram
ASCII diagram or numbered step list. This is the "what connects to what" layer, deliberately separate from the equations — most implementation bugs live here, in signal-flow mistakes rather than arithmetic ones.

For composite algorithms, show data types flowing along each edge (frame vs sample, time-domain vs frequency-domain, magnitude vs complex spectrum).

## Reference implementations examined
For each of ≥2 reference implementations:

### Ref N — <source name>
- **Location**: full URL + commit hash / version tag, plus date accessed
- **What I learned that wasn't in the paper**: implementation details — start-up, denormal handling, edge cases, parameter defaults used in production, numerical reformulations
- **Where it agrees with the primary source**: specific points of agreement
- **Where it disagrees / differs**: concrete discrepancies with concrete positions on who's right, with reasoning
- **Trust level**: HIGH/MEDIUM/LOW, and for which aspects — you can trust a repo's topology while distrusting its exact coefficients, or vice versa

## Disagreements resolved
Aggregate section listing every contradiction you found between sources (paper vs ref impl A, ref impl A vs ref impl B, paper's body vs paper's appendix) and how you resolved each. Every unresolved disagreement is a bug waiting to happen; list them as Open Questions.

## Complexity & resource profile
- Computational complexity: O(?) per sample / per frame / per FFT block
- FLOPS estimate (multiplies + adds) per sample
- Memory footprint (bytes, in terms of parameters like block size, delay length, filter order)
- Latency (samples of algorithmic delay, distinct from buffer/group delay)
- Real-time suitability (block size vs deadline)
- Platform notes if relevant (vectorization opportunities, cache behavior, branch patterns)

## Domain-specific quality criteria
What does "good" mean for this algorithm, in the terms of its domain? Examples:
- Reverb: RT60 accuracy; flat late-tail spectrum for neutral rooms; echo density (Jot's Diffusion quality); modal density; EDR curve shape
- Pitch detection: raw pitch accuracy (±50 cent), raw chroma accuracy, voiced F1, octave error rate, polyphonic behavior
- Compressor: static gain curve accuracy; attack/release shape; distortion products; sidechain rejection
- Filter: -3dB at stated cutoff; stopband attenuation; passband ripple; phase linearity deviation; group delay flatness
- NN model: task-specific metric (WER, PESQ, STOI, MOS)

Name 2-5 objective or perceptual metrics by which this implementation's output will be judged, with expected numerical ranges or qualitative signatures.

## Known pitfalls (from papers, reference implementations, personal experience, and issue trackers)
- Sign conventions — especially where different traditions use opposite signs
- Sample-rate-dependent constants — list every one derived from SR
- Delay compensation / group delay — especially for multi-stage algorithms
- Denormal / NaN / Inf handling
- Fixed-point vs floating-point behavior divergence
- Paper typos (yes, famous papers have typos — Dattorro 1997's plate paper is a known example)
- Known buggy reference implementations that propagate error
- Numerical conditioning — when does the algorithm become unstable or lose accuracy?

## Parameter interpretation

### Runtime (knob-twistable) parameters
| Name | Range | Unit | Controls | Default | Sensitivity |
|------|-------|------|----------|---------|-------------|
| ... |

Sensitivity column: is this parameter dominant (changes output character), moderate (noticeable but not character-changing), or cosmetic (fine-tuning)?

### Compile-time / structural parameters
| Name | Valid range | Why compile-time |
|------|-------------|------------------|

## Parameter sensitivity analysis
Which 1–3 parameters dominate the output? For each:
- What does sweeping it from min to max do perceptually/numerically?
- Are there pathological values (stability, aliasing, overflow)?
- What's a "safe by default" setting?

This is the section that lets a future engineer debug reports like "the thing sounds dark" without re-reading the paper.

## Failure mode gallery
What does "wrong" look like? Describe at least 3 failure modes, for each:
- **Symptom**: observable behavior (audio artifact, visual artifact, numerical signature)
- **Cause**: specific implementation error (wrong sign, missing stage, bad initial condition, etc.)
- **Detection**: automated check that would catch this (test signal + assertion)

This transforms the brief from a spec into a debugging aid.

## Validation strategy
Concrete test signals → expected response. Each test must be objectively checkable.

- Impulse → <expected impulse response signature, qualitatively and numerically>
- Step → <expected step response>
- Sine sweep (20 Hz – 20 kHz, if audio) → <expected magnitude / phase response shape>
- DC → <expected output, zero or finite>
- Silence → <expected output after settling>
- Domain-specific test signals: chirp, multi-sine, pink noise, known-pitch vowels, impulse responses from standard rooms, etc.
- Regression signals: any that have caught prior bugs (if the algorithm has a project history, list the commits that fixed and the signals they added)

For each test, name the assertion: "RT60 measured from EDR should be within 10% of configured RT60" is actionable; "sounds roomy" is not.

## Parameter defaults that work
Opinionated defaults for the common case, drawn from reference implementations and the paper. Not just "what the knob does" (covered above) but "what should the knob be when the musician first instantiates this".

## Open questions
Anything unresolved after research. Decisions for the user, not guesses for you. Be specific — each open question should be answerable with a single choice or parameter value.
```

---

### Phase 3 — Gate (hard stop)

After saving the brief, present a concise summary to the user and request explicit approval. Example:

> 研究简报已落盘到 `docs/algorithms/<name>.md` (约 <N> 行)。要点：
>
> - 主源：<paper citation>
> - 历史定位：<one line from Historical context section>
> - 核心洞察：<one line from Intuition section>
> - 参考实现：<ref impl 1 (trust level)>; <ref impl 2 (trust level)>
> - 已解决的主要分歧：<2-3 bullet list of disagreements resolved>
> - 待决策（需要你拍板）：<numbered list of open questions>
> - 复杂度/延迟：<one line>
> - 预期声音/结果特征：<one line from Domain-specific quality criteria>
>
> 确认无误后再开始写代码。

Do not write implementation code until the user approves. "Approves" means an explicit yes, OK, 确认, 可以, 开始写, or equivalent. Silence, "looks good", or answering a different question is NOT approval.

If the user redirects — a missing source, a wrong equation, a preferred variant, a request for more depth on a specific section — revise the brief on disk, then re-present and re-ask. Iterate as many times as needed; the brief is the deliverable for this phase.

**The gate is non-negotiable.** The purpose of this entire skill is to make the user's understanding of what you think the algorithm *is* precede any code written against that understanding. Skipping the gate defeats the skill.

---

### Phase 4 — Implement

Write the code with the brief open in another window. Every coefficient, sign, delay length, initial condition, topology choice, and parameter default should trace to something in the brief — an equation, a reference impl citation, or a noted deliberate deviation. If you cannot point to such a source, do not write that constant.

If during implementation you find a question not covered in the brief:
- If it's mechanical (choice of variable name, private helper structure): make it.
- If it changes user-visible behavior (default value, failure mode, edge case): stop, update the brief, and either proceed silently (if trivial) or re-gate (if substantial).

Prefer implementation style that **reads like the brief**: comments reference equation numbers; topological stages are clearly delimited; parameter setters echo the brief's parameter table.

Write validation tests from Phase 2 / Validation strategy. Every named test in the brief should correspond to a test in code.

---

### Phase 5 — Blind sub-agent review (independent second reading, NOT confirmation)

After the implementation compiles and basic tests pass, dispatch an independent review using the Agent tool with `subagent_type: "general-purpose"` (or a code-reviewer agent if available).

**This phase is NOT "let a second agent check our work." It is "let a second agent independently read the primary sources, form its own interpretation of the algorithm, and then compare that interpretation to the code."** A reviewer that accepts our brief as ground truth can only catch surface-level issues. A reviewer that independently interprets the primary paper can catch the entire class of "our brief misread the paper" errors — exactly the class this skill exists to prevent.

The reviewer gets the **same raw materials we had access to**, but NOT our interpretation of them.

**GIVE the sub-agent** (default: D-mode, "raw sources only"):
- Algorithm's canonical name
- Path(s) to the implementation file(s)
- DOI + stable URL of the primary paper (extracted from the brief's "Primary source" section — but do NOT include the brief itself)
- GitHub URLs + commit/version refs for ≥2 reference implementations (extracted from the brief's "Reference implementations examined" section)
- Optional: URL of a survey/review paper (from "Historical context" section)
- Project design rules file (CLAUDE.md or equivalent) — the reviewer needs language/style conventions to evaluate the code

**DO NOT give the sub-agent**:
- **The research brief itself — this is the most important exclusion.** The brief is our interpretation of the sources; the reviewer's job is to form its own. Passing the brief contaminates the review.
- Your conversation history, design decisions, or specific concerns
- The user's preferences expressed during this session
- Any summary of the algorithm beyond its canonical name

The reviewer MUST do its own research — read the paper, read the reference implementations, form its own mental model. This takes time (≈10–20 minutes for a moderate algorithm) and tokens. **Pay the cost; this is where the skill earns its keep.** A reviewer that just verifies our brief delivers near-zero independent signal.

**Prompt template**:

> You are an independent DSP / domain-specific reviewer. You have NO context on prior work, reasoning, or decisions. Your job is to form your own interpretation of <algorithm name> by reading primary sources, then evaluate a colleague's implementation against that independently-formed interpretation.
>
> PRIMARY SOURCES (your authority):
> - Paper: <DOI and stable URL>
> - Additional context / related survey (optional): <URL>
>
> REFERENCE IMPLEMENTATIONS (read critically — they are examples, not authorities):
> - <ref impl 1 — repo URL + commit/version>
> - <ref impl 2 — repo URL + commit/version>
>
> SUBJECT OF REVIEW:
> - Implementation: <file paths>
> - Project conventions: <path to CLAUDE.md>
>
> PROCESS (do in order, do NOT skip):
> 1. Read the primary paper (WebFetch). Form your own mental model.
> 2. Read the reference implementations (WebFetch). Note agreements/disagreements with paper and each other.
> 3. Read the subject code. Compare against your mental model from step 1.
>
> QUESTIONS TO ANSWER (concrete, file:line references):
> 1. Does the code match the algorithm's specification as you understand it from the paper?
> 2. Are coefficients, sign conventions, and initial conditions consistent with the paper's equations and reference implementations?
> 3. Any stages missing, simplified, or substituted?
> 4. Numerical pitfalls (denormals, overflow, divide-by-zero, SR coupling, initial-condition races) unhandled?
> 5. Confidence (0–100%) and the single highest-value next check to raise it.
>
> RULES:
> - Your training knowledge is a heuristic starting point, NOT ground truth. Paper + reference code you read are the authorities.
> - If paper and a reference implementation disagree, call it out and state your lean with reasoning.
> - Classify each finding: BUG (functional error) / CONCERN (smell worth investigating) / NOTE (observation, no action).
> - Do NOT propose fixes; report observations.
> - If a source is paywalled or inaccessible, flag it and work with what you can access.
> - Sycophancy wastes time. State correctness clearly; surface issues intact.

**Reporting back**: Report the sub-agent's findings to the user verbatim or near-verbatim. Do not filter, defend, or dismiss. If the sub-agent flags something, **prefer to investigate rather than dismiss**. A blind reviewer arriving at a finding the main agent missed is the signal this phase exists to produce.

**C-mode fast path (exception, use sparingly)**:
For exceptionally well-established textbook algorithms (biquad, FFT radix-2, FIR windowing) where primary sources are universally available in reviewer training AND the brief was written with high confidence, you MAY pass the brief as a "variant disambiguator" in place of raw sources. This trades away independence for speed. Default is D-mode; C-mode requires explicit justification ("algorithm is textbook-standard, brief has no ambiguities").

**BOTH modes for high-stakes algorithms**:
For novel composites, safety-critical work, or where the brief's interpretation of the paper was uncertain, run Phase 5 in both D-mode AND C-mode in parallel. Convergence across modes = strong validation. Divergence = the user's to adjudicate — and valuable information either way.

---

## Output persistence & cross-device reuse

Briefs in `docs/algorithms/` are checked into git deliberately. They are institutional memory — on future devices, future sessions, or when onboarding a collaborator, these briefs are a head-start, not a redo.

When starting a new algorithm in a project that already has `docs/algorithms/<something-related>.md`, read those first. If a previous brief covers exactly the algorithm being implemented, you are NOT exempt from the workflow — but start from the existing brief, verify it's still accurate, update sections that have aged, and proceed through gate → implement.

If a brief exists but was written to a shallower standard than this skill demands (e.g., ~150 lines with no Historical context, no Derivation, no Failure mode gallery), treat deepening it as part of Phase 2 before gating.

## Failure modes to watch for

- **Memory leaking into the brief.** If you write an equation or coefficient without a source attribution, stop and find one. "I'm pretty sure it's …" is never a source.
- **Rubber-stamping reference code.** A popular reference repo is not proof of correctness. State an assessment.
- **Treating length as a proxy for depth.** A 500-line brief that is padding is worse than a 250-line brief with real content. Depth is the goal; length is a byproduct.
- **Skipping the gate because "this one is simple."** The class of bug this skill prevents is precisely the one you didn't see coming. Simple-looking algorithms are where memory errors hide best.
- **Writing the brief after the code.** The brief is a specification, not a post-hoc justification. Code must follow the brief, not the other way around.
- **Feeding the review sub-agent context.** Giving it your reasoning turns an independent reviewer into a confirmer. Keep it blind.
- **Soft-pedaling sub-agent findings.** If the blind review raises a concern, surface it intact. The user decides what's a real problem.
- **Deferring the "Failure mode gallery" section** because it feels speculative. It isn't speculative; it's triage infrastructure for future debugging.
- **Skipping Source kind 3 (survey / context).** This is the section a busy engineer cuts first and regrets later when their algorithm turns out to have been superseded five years ago.

## A note on naming files

Algorithm brief filenames should be stable and descriptive: `dattorro_plate_reverb.md`, `jot_fdn.md`, `yin_pitch.md`, `pyin_pitch.md`, `karplus_strong.md`, `rbj_biquad_cookbook.md`, `linkwitz_riley_crossover.md`. Prefer snake_case. Include the variant or author when it distinguishes (`schroeder_allpass_reverb.md` vs `moorer_reverb.md`; `yin_pitch.md` vs `pyin_pitch.md`).

If you're implementing a variant of an already-documented algorithm, either:
- Extend the existing brief with a "## Variants" section, cross-referencing the new variant
- Create a new brief `<base_name>_<variant>.md` that cross-references the base

Don't fragment. Don't duplicate.

## For the researcher reading this in six months

If you are the user's future self, or a collaborator onboarding, the briefs in `docs/algorithms/` are your starting point. They were written with the discipline this skill enforces, which means: they contain the derivations, the context, the failure modes, and the resolved contradictions that the papers alone don't. Use them. Update them when reality diverges. Write a new one when a new algorithm enters the project.

The effort to maintain them is the whole point of the workflow.
