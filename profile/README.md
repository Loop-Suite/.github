# Loop-Suite

**Independent lens review → anonymized cross-verification (discourse) → deterministic verdict.** One execution pattern, ported across domains as a family of Rust CLIs.

Each tool has multiple expert personas review content independently, cross-check each other's judgments anonymously (no author identity, to avoid authority bias), and then compute the score and pass/fail outcome with local deterministic logic — not the LLM. The *aggregation arithmetic* (severity weights, confidence thresholds, PASS/BLOCK cutoffs) is always plain code, not an LLM call. But its inputs — finding severity, discourse resolution, citation/verification status — are themselves LLM-produced judgments, not independently re-verified facts. Treat "deterministic verdict" as "deterministic arithmetic over model judgments," not as external validation.

## Why this pattern

A single LLM call reviewing its own (or someone else's) work has three well-known failure modes: it doesn't know what it doesn't know, it tends to agree with whatever framing the prompt implies, and its confidence doesn't track its actual accuracy. Multiplying calls doesn't fix this by itself — three correlated calls to the same model are still one opinion asked three times.

Loop-Suite's answer, applied consistently across every tool in this org:

- **Split judgment from arithmetic.** An LLM decides *what's true* (a finding, a ranking, a category). Plain Rust code decides *what that adds up to* (a score, a PASS/FAIL, a verdict). If a bug is in the scoring, it's a reproducible code bug, not a re-roll-and-hope prompt problem.
- **Force independence before consensus.** Reviewers don't see each other's names or, in most tools, each other's opinions before rendering their own verdict. Agreement has to survive being computed from anonymous, uncorrelated inputs — not from social pressure to converge with whoever answered first.
- **Never trust a verdict you can't regenerate.** Every tool's state (`state.json`) captures enough to re-run deterministic checks without another LLM call (`validate`-style commands) and to feed prior findings back into a new round (`--prior` / `refine`) instead of starting blind every time.

The specifics — how many personas, whether discourse is sequential debate or independent blind voting, what counts as a valid challenge — differ by domain. See [Pipelines](#pipelines) below for what each tool actually does, not just the abstract shape.

```mermaid
flowchart LR
    In["Input\n(diff / content / brief)"] --> Sel["Lens Selection\nLLM picks 1–3 personas"]

    Sel --> P1["Persona A\nindependent review"]
    Sel --> P2["Persona B\nindependent review"]
    Sel --> P3["Persona N\nindependent review"]

    P1 --> Anon["Anonymize\n(strip reviewer identity)"]
    P2 --> Anon
    P3 --> Anon

    Anon --> Dis["Discourse\nAGREE · CHALLENGE · CONNECT · SURFACE"]
    Dis --> Res["Resolution\nCONFIRMED / REJECTED / MERGED / UNCERTAIN"]

    Res --> Det["Deterministic Scoring\nlocal code, no LLM"]
    Det --> Out["Verdict + Report"]

    Out -.->|"--prior next round"| Sel
```

## Getting started

Every tool is an independent Rust binary — clone the one you need, there's no shared crate or monorepo build:

```bash
git clone https://github.com/Loop-Suite/<name>.git
cd <name>
cargo build --release
```

All of them default to the `claude` CLI as the LLM backend (a subprocess call, `claude -p --output-format json` — no separate API key needed if you already use Claude Code), with an OpenRouter backend as an alternative or, in `icon-loop` and `store-creative-loop`, as a second panel member for judge diversity. Each repo's own `README.md` has the actual subcommands and a real example invocation — start there.

## Projects

| Repo | Domain | Notes |
|---|---|---|
| **[Code-Review-Loop](https://github.com/Loop-Suite/Code-Review-Loop)** | PR diff code review | public. The original Rust CLI — where this pattern started. Also bundles a reproducible benchmark for the related `full-review` Claude Code skill under [`harness/`](https://github.com/Loop-Suite/Code-Review-Loop/tree/main/harness) (Node-based, separate from the discourse pipeline). |
| **[Marketing-Loop](https://github.com/Loop-Suite/Marketing-Loop)** | Marketing content review (ad copy, landing pages, email, social, blog) | public. Full `src/` implementation (18 Rust files) — `review`/`describe`/`improve`/`ask` subcommands, 7-persona pool, discourse cross-check, deterministic policy gates. Reviewed across multiple rounds (14 issues found and fixed), 73 unit tests, CI, `CHANGELOG.md`, tagged `v0.1.1`. |
| **[Bizplan-Loop](https://github.com/Loop-Suite/Bizplan-Loop)** | Business plan generation (N drafts → rubric scoring → regenerate) | public, a variant without discourse |
| **[ASO-Loop](https://github.com/Loop-Suite/ASO-Loop)** | App Store Optimization listing copy (title/subtitle/keywords/description) | public, Bizplan-Loop variant (no discourse). Keyword normalization/dedup logic ported from semihcihan/App-Store-Optimization-CLI (MIT). |
| **[GEO-Loop](https://github.com/Loop-Suite/GEO-Loop)** | Generative Engine Optimization content (citability for ChatGPT/Perplexity/Google AI Overviews) | public, Bizplan-Loop variant (no discourse). llms.txt validation ported from Auriti-Labs/geo-optimizer-skill (MIT); JSON-LD scaffolding from ai-search-guru/getcito (MIT); FAQPage/Article required-field checks implemented from Google Search Central's structured-data guidelines (not a code port). |
| **[SEO-Loop](https://github.com/Loop-Suite/SEO-Loop)** | SEO content copy (title/meta/heading structure/E-E-A-T signals) | public, Bizplan-Loop variant (no discourse). Complementary to the (non-Loop-Suite) `seo-reference-library` skill — that one audits existing sites, this one generates and scores new copy. Flesch readability logic ported from IamRamgarhia/BlogPilot (MIT); paragraph-length check idea from CyberCraftBD/power-seo (MIT). |
| **[Research-Loop](https://github.com/Loop-Suite/Research-Loop)** | Market/competitor research document validation | public. Motivated by a real multi-round POS-competitor research doc; catches quantitative-vs-qualitative disagreement, a subject's own marketing dominating its citations, incentivized reviews, numeric drift across drafts, and prior conclusions overturned by newer evidence (`REVERSED` status). Source-level re-verification of GPT Researcher/company-research-agent/MetaGPT is in its evidence survey. |
| **[Secretscan-Loop](https://github.com/Loop-Suite/Secretscan-Loop)** | Pre-push/pre-publish secret-scanner-finding triage | public. Domain already has mature deterministic scanners (gitleaks/trufflehog) — the discourse layer triages *their* output. Masking was rewritten to a byte-range-merge algorithm (capture-group only, safe under partial overlap); `hard_verified` (TruffleHog `Verified=true` or private-key shape) forces `BLOCK` before discourse even runs, regardless of LLM judgment. Reviewed across 4 rounds (9/9 issues closed), 24 unit tests, CI, tagged `v0.1.0`. |
| **[Icon-Loop](https://github.com/Loop-Suite/Icon-Loop)** | App icon (vector) design | public. Discourse variant — critics are fully independent and blind (no sequential AGREE/CHALLENGE exposure), aggregated with a deterministic Borda count instead; also flags unanimous agreement (possible collusion) and surfaces minority opinions a naive aggregate would bury. Critic backends can mix claude/openrouter so the panel isn't just the same model called N times. Rendering is pure Rust (resvg/usvg/tiny-skia, no external binary), and policy.rs checks canvas containment/palette/small-size legibility against the actual rendered pixels, not the raw SVG source. |
| **[store-creative-loop](https://github.com/Loop-Suite/store-creative-loop)** | App-store screenshot sets and feature graphics | public. Generates real Apple/Google phone, tablet, and feature-graphic PNGs from ordered, device-specific raw captures. Each selected audience/intent segment receives explicit product-led, outcome-led, and trust-led hypotheses; local Rust code renders exact target canvases and blocks invalid assets plus unsupported ratings, rankings, awards, percentages, or guarantees. Blind critics select an offline direction and feed concrete risks into the next round; `experiment.md` reserves conversion claims for Apple PPO or Google Store Listing Experiments. |

## Pipelines

The shared diagram above is the abstract shape. Each project bends it differently — module names, persona pools, and what counts as a valid `CHALLENGE` are all domain-specific. One diagram per repo, grounded in its actual source:

### Code-Review-Loop

The original. 12-step pipeline, PR diff in, review report out.

```mermaid
flowchart LR
    A["PR diff + spec"] --> B["input.rs: normalize diff"]
    B --> C["lens.rs: select 3-5 lenses"]
    C --> D["Independent review<br/>Fowler · Ousterhout · Beck · Metz · Dodds · Khorikov · Hickey"]
    D --> E["discourse.rs: AGREE / CHALLENGE / CONNECT / SURFACE"]
    E --> F["requirements.rs + policy.rs<br/>(semgrep, if on PATH)"]
    F --> G["quantify.rs: score + verdict"]
    G --> H["report.md + state.json"]
    H -.->|"--prior next round"| B
```

### codedesign-loop

Same shape, one stage earlier — reviews a design/architecture brief before any code exists.

```mermaid
flowchart LR
    A["brief + repo-context + constraints"] --> B["discover.rs: problem, users, non-goals"]
    B --> C["requirements.rs: REQ ids + acceptance criteria"]
    C --> D["lens.rs: independent design lenses"]
    D --> E["discourse.rs: anonymized cross-check"]
    E --> F["contract.rs: components, interfaces, ADRs"]
    F --> G["tasks.rs: traceable task breakdown"]
    G --> H["policy.rs: deterministic quality gates"]
    H --> I["quantify.rs: score + verdict"]
    I --> J["design.md / tasks.md / report.md / state.json"]
    J -.->|"refine with feedback"| B
```

### marketing-loop

Fully implemented — content review with a 7-persona pool, 1-3 selected per run plus `claims_compliance` always on.

```mermaid
flowchart LR
    A["content + brief + style guide"] --> B["lens.rs: pick 1-3 optional of 7 personas<br/>+ claims_compliance always-on"]
    B --> C["Independent review<br/>Handley · Fishkin · Tushnet · Laja · Godin · Dunford · Ogilvy"]
    C --> D["discourse.rs: cross-examination"]
    D --> E["checks.rs: banned words, disclaimers, length limits"]
    E --> F["requirements.rs: brief verification"]
    F --> G["quantify.rs: score + verdict"]
    G --> H["report.md<br/>(+ optional human-voice rewrite)"]
```

### bizplan-loop

The one variant with no discourse — generate N drafts, score, keep the best, optionally loop.

```mermaid
flowchart LR
    A["idea + spec"] --> B["generate.rs: N angle-varied drafts"]
    B --> C["checks.rs: deterministic section/length/citation checks"]
    C --> D["score.rs: LLM rubric<br/>multiple judge models/rounds"]
    D --> E["trimmed-mean aggregation per criterion"]
    E --> F{"loop mode?"}
    F -->|"gen"| G["best.md + ranked runs"]
    F -->|"loop, target score"| H["feedback → regenerate<br/>until target / max-iter / patience"]
    H --> B
    G --> I["held-out gate model re-scores<br/>first vs. best (reward-hacking check)"]
```

### aso-loop

Same no-discourse shape as bizplan-loop, ported to app-store listing copy.

```mermaid
flowchart LR
    A["brief + spec (apple/google)"] --> B["generate.rs: N angle-varied drafts"]
    B --> C["checks.rs: char/byte limits, keyword coverage,<br/>cross-field dedup, banned/superlative terms"]
    C --> D["score.rs: LLM rubric<br/>multiple judge models/rounds"]
    D --> E["trimmed-mean aggregation per criterion"]
    E --> F{"loop mode?"}
    F -->|"gen"| G["best.md + ranked runs"]
    F -->|"loop, target score"| H["feedback → regenerate<br/>until target / max-iter"]
    H --> B
    G --> I["held-out gate model re-scores<br/>first vs. best (reward-hacking check)"]
```

### geo-loop

```mermaid
flowchart LR
    A["idea + spec"] --> B["generate.rs: N angle-varied drafts"]
    B --> C["checks.rs: answer-summary, statistics/sources,<br/>FAQ, JSON-LD/schema.org fields, llms.txt"]
    C --> D["score.rs: LLM rubric<br/>multiple judge models/rounds"]
    D --> E["trimmed-mean aggregation per criterion"]
    E --> F{"loop mode?"}
    F -->|"gen"| G["best.md + ranked runs"]
    F -->|"loop, target score"| H["feedback → regenerate<br/>until target / max-iter"]
    H --> B
    G --> I["held-out gate model re-scores<br/>first vs. best (reward-hacking check)"]
```

### seo-loop

```mermaid
flowchart LR
    A["brief + spec"] --> B["generate.rs: N angle-varied drafts"]
    B --> C["checks.rs: title/meta length, heading hierarchy,<br/>keyword placement, Flesch readability, link/citation counts"]
    C --> D["score.rs: LLM rubric<br/>multiple judge models/rounds"]
    D --> E["trimmed-mean aggregation per criterion"]
    E --> F{"loop mode?"}
    F -->|"gen"| G["best.md + ranked runs"]
    F -->|"loop, target score"| H["feedback → regenerate<br/>until target / max-iter"]
    H --> B
    G --> I["held-out gate model re-scores<br/>first vs. best (reward-hacking check)"]
```

### research-loop

```mermaid
flowchart LR
    A["research document + brief"] --> B["checks.rs: 7 deterministic checks"]
    A --> C["lens.rs: 7 personas<br/>Porter · Damodaran · McKenzie · Orosz · Doctorow · Grant · boyd"]
    C --> D["discourse.rs: AGREE / CHALLENGE / CONNECT / SURFACE<br/>CHALLENGE narrowed to re-measurement evidence"]
    D --> E["requirements.rs: coverage_gaps"]
    B --> F["quantify.rs: PASS / REVISE"]
    E --> F
    F --> G["report.md + state.json"]
    G -.->|"--prior (tracks REVERSED too)"| A
```

### secretscan-loop

```mermaid
flowchart LR
    A["target directory"] --> B["scanners.rs: builtin scanner<br/>+ gitleaks/trufflehog if on PATH"]
    B --> C["masked candidate list<br/>(byte-range-merge masking, safe under partial overlap)"]
    C --> D["lens.rs: 7 personas<br/>Hunt · Moore · Janca · Hightower · Schneier · Moussouris · Herold"]
    D --> E["discourse.rs: fixture-vs-real-format evidence only"]
    E --> F["checks.rs: .gitignore coverage, tracked-sensitive-files"]
    F --> G["quantify.rs: BLOCK / WARN / PASS"]
    G --> H["report.md + state.json"]
    H -.->|"--prior (tracks ROTATED too)"| A
```

### icon-loop

The one variant where discourse is fully independent instead of sequential — critics never see each other's rulings, so the AGREE/CHALLENGE debate step is replaced by parallel blind calls plus deterministic vote aggregation. In practice: a persona was confident its own triangle-with-eye design wouldn't be misread, and every independent critic read it as a mountain/spark shape anyway — the kind of mismatch deterministic geometry checks can't see, and that sequential discourse (where the first opinion can anchor the rest) is more likely to paper over than catch.

```mermaid
flowchart LR
    A["persona spec + brief"] --> B["lens.rs: independent persona<br/>SVG generation"]
    B --> C["render.rs: resvg/usvg/tiny-skia<br/>SVG to PNG, no external binary"]
    C --> D["policy.rs: deterministic gates<br/>containment / palette / small-size legibility"]
    D --> E["blind anonymization<br/>candidate_N filenames"]
    E --> F["discourse.rs: N critics,<br/>fully independent, no cross-exposure<br/>critic_backends mixes claude/openrouter"]
    F --> G["quantify.rs: Borda count<br/>+ collusion warning + minority opinions"]
    G --> H["report.md + state.json"]
    H -.->|"refine --prior<br/>(--no-critique for ablation)"| B
```

### store-creative-loop

This is a segment-aware screenshot generation-and-learning loop. It turns ordered, device-specific raw captures into multi-device store PNGs, blocks objectively invalid exports and unsupported trust claims, evaluates product/outcome/trust hypotheses blindly, and sends the selected plan plus observed weaknesses into the next rendering round. Existing external renderers can still enter directly at validation or review. Model-assisted selection and market validation remain two explicitly different stages.

```mermaid
flowchart LR
    A["device-specific raw captures + product truth<br/>segment · intent · target spec"] --> B["generation.rs: product / outcome / trust<br/>hypothesis id · copy · order · layout"]
    B --> C["deterministic renderer<br/>UI-dominant phone / tablet / feature PNGs"]
    C --> D["policy gates<br/>size / alpha / hashes / unsupported claims"]
    D --> E["contact_sheet.rs<br/>anonymous per-target thumbnail sheets"]
    E --> F["critique.rs: independent visual critics<br/>cyclic order · no cross-exposure"]
    F --> G["quantify.rs: Borda + criterion means<br/>dissent + corroborated risks"]
    G --> H["winner plan + concrete feedback"]
    H -.->|"next generation iteration"| B
    H --> I["final PNG set + experiment.md<br/>offline recommendation only"]
```

### Code-Review-Loop → `harness/` (the benchmark — different shape entirely)

Not the Rust pipeline above. A Node.js harness, bundled under `harness/` in the same repo, that benchmarks the separate `full-review` Claude Code skill against known-mutation fixtures.

```mermaid
flowchart LR
    A["review-target fixture<br/>+ injected mutations, vs. clean-reference"] --> B["claude-invocation.mjs<br/>runs /full-review via claude CLI"]
    B --> C["extract-claude-result.mjs"]
    C --> D["verify-results.mjs / verify-mutations.mjs<br/>vs. ground-truth.json / expected-failures.json"]
    D --> E["evaluate.mjs: precision / recall / cost"]
    E --> F["audit-transcript.mjs"]
    F --> G["oracle.test.mjs: reproducible pass/fail"]
```

## Shared principles

- **Rust CLI**, defaulting to the Claude Code CLI (`claude -p --output-format json`) as a subprocess backend — no separate API key required. An OpenRouter backend is also supported.
- **Deterministic logic and LLM judgment are split at the code level.** Scoring, policy checks, and verdicts are computed locally and deterministically; the LLM only handles judgment-dependent steps like lens selection, review, discourse, and requirement verification.
- **Anonymized cross-verification**: reviewer identity is stripped before discourse, so agreement/disagreement is forced to run on claims and evidence rather than persona authority.
- Re-running with `--prior` tracks prior observations instead of starting from a clean slate (except bizplan-loop and its aso-loop/geo-loop/seo-loop siblings, which are generate-and-score rather than review-based). Domain-specific state vocabularies reflect what the evidence can actually establish: research-loop adds `REVERSED` (a prior conclusion overturned by newer evidence, not just updated), secretscan-loop adds `ROTATED` (the string is still physically present but the credential itself was rotated/revoked), and store-creative-loop uses `STILL_OPEN` / `NEW` / `NOT_REOBSERVED`—deliberately never upgrading a finding to `FIXED` merely because a new model panel did not reproduce it.
- Where a project reuses logic from outside Loop-Suite (e.g. aso-loop/geo-loop/seo-loop porting small algorithms from MIT-licensed tools, or citing public guidelines like Google Search Central's structured-data docs), the source is cited in that repo's `NOTICE`/`README.md` — never copied from code without a compatible license.

## What "deterministic verdict" doesn't mean

Worth repeating from the top, because it's the most common misread of this pattern: the score/verdict math being deterministic code doesn't make the *verdict itself* independently verified fact. The inputs to that math — a finding's severity, whether a CHALLENGE was resolved, whether a citation checks out — are still LLM judgments. Determinism buys you *reproducibility* (the same findings always score the same way) and *auditability* (you can read the scoring code and know exactly why a verdict landed where it did) — not ground truth. Read a `report.md`'s findings as claims to spot-check, not as settled facts.

## License

Every repo listed above is Apache-2.0 — see each repo's own `LICENSE`.

---

[Code-Review-Loop](https://github.com/Loop-Suite/Code-Review-Loop), [Marketing-Loop](https://github.com/Loop-Suite/Marketing-Loop), [Bizplan-Loop](https://github.com/Loop-Suite/Bizplan-Loop), [ASO-Loop](https://github.com/Loop-Suite/ASO-Loop), [GEO-Loop](https://github.com/Loop-Suite/GEO-Loop), [SEO-Loop](https://github.com/Loop-Suite/SEO-Loop), [Research-Loop](https://github.com/Loop-Suite/Research-Loop), [Secretscan-Loop](https://github.com/Loop-Suite/Secretscan-Loop), [Icon-Loop](https://github.com/Loop-Suite/Icon-Loop), and [store-creative-loop](https://github.com/Loop-Suite/store-creative-loop) are public today. One more (pre-code design/architecture review domain) stays private for now.
