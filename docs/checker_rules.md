# Checker rules

The Checker scores each generated chapter against eight rules. Any rule failing returns the chapter to the Writer with the failure reason concatenated into the next prompt; the chapter is regenerated at a slightly higher temperature. Up to three attempts; if all three fail, the highest-scoring attempt is kept.

This document describes the eight rules as implemented in `lora/colab_full_pipeline.ipynb` (cell 6). All rules are pure-Python checks against the generated text; none of them require the source novels at inference time.

## Rule 1 — Cross-universe contamination

The fanfic training corpus has a small amount of material from a different universe (Xueying Lord). We want zero leakage of Xueying-specific terminology into Tunshi generation.

```python
XUEYING_WORDS = ['开辟境', '东伯雪鹰', '雷霆领域', '雪鹰',
                 '黑沼', '雷神军', '灵剑山']

for w in XUEYING_WORDS:
    if w in text:
        fail(f"cross-universe term '{w}' appeared {count} times")
```

Each term is universe-unique with zero legitimate use in a Tunshi fanfic. Shared neutral vocabulary (e.g. `领主`, which exists in both universes) is intentionally not in the blacklist.

## Rule 2 — AI-style sentence fragmentation

Generic LLMs default to a poetry-like cadence with many short period-terminated lines. This is the opposite of the long action-driven sentences we want.

```python
AI_SHORT_RE = re.compile(r'(^[^。\n]{1,8}。\n){3,}', re.MULTILINE)
if AI_SHORT_RE.search(text):
    fail("AI-style fragmentation: 3+ consecutive lines of ≤8 chars period-terminated")
```

A single short sentence is fine; three or more in a row triggers the rule.

## Rule 3 — Protagonist independence

A common failure mode after LoRA on a fanfic corpus is regression toward the canonical protagonist. The Checker requires that the user-supplied protagonist appears prominently and that the canonical lead (`罗峰`) does not steal the spotlight.

```python
name = setup['protagonist_name']
if text.count(name) < 8:
    fail("protagonist appears < 8 times")
if text.count('罗峰') > 5:
    fail("canonical lead 罗峰 appears > 5 times")
```

The thresholds are tuned to catch obvious "Luo Feng cameo takeover" failures while allowing brief mentions.

## Rule 4 — Web-novel paragraph format

Chinese web novels use full-width double-space (`　　`, U+3000 ×2) for paragraph indentation. Generic LLM output uses half-width spaces or no indentation at all. The ratio of indented paragraphs to total newlines is a strong style signal.

```python
fw_pairs = text.count('　　')
lines = text.count('\n')
ratio = fw_pairs / max(1, lines)
if ratio < 0.2:
    fail(f"paragraph format ratio {ratio:.2f} < 0.2")
```

A 0.2 threshold corresponds to roughly one indented paragraph every five lines, which is the minimum observed in real web-novel chapters.

## Rule 5 — Chapter length

A chapter that's too short usually indicates the model gave up early or fell into a degenerate state.

```python
if len(text) < 1200:
    fail(f"chapter length {len(text)} < 1200 chars")
```

The 1200-character floor is well below the target (~2500) but catches genuine truncation. We deliberately do not enforce an upper bound: longer chapters are not penalised here.

## Rule 6 — Web-novel meta-noise residue

The LoRA was trained on a fanworks corpus that, despite cleaning, retained traces of author asides, reader-facing solicitations, serialisation markers, and forum sign-off lines. The model learns these and re-emits them. Rule 6 catches the most common patterns:

```python
NOISE_PATTERNS = [
    r'未完待续', r'还在连载', r'连载中', r'下一章',
    r'求收藏', r'求推荐', r'求订阅', r'求月票', r'求打赏',
    r'感谢.{0,10}打赏', r'感谢.{0,10}书友', r'感谢.{0,10}读者',
    r'作者的话', r'作者注', r'读者.{0,5}说', r'读者评论',
    r'本章完', r'章节内容', r'书友们', r'本书读者群',
    r'ixdzs', r'爱下电', r'txt80',
]
if NOISE_RE.search(text):
    fail("meta-noise residue: " + matched terms)
```

A clean chapter should be pure narrative; any of these phrases means the model is producing author-mode text rather than story-mode text.

## Rule 7 — Placeholder echo

If a prompt template contains square-bracket or full-width-bracket scaffolding like `[书名]` or `【主角】`, an under-trained model will copy the brackets into its output instead of filling them in. Rule 7 catches this:

```python
PLACEHOLDER_RE = re.compile(r'[【\[][一-鿿]{1,8}[】\]]')
if PLACEHOLDER_RE.search(text):
    fail("placeholder echo: e.g. 【主角】, [书名]")
```

The Outline prompt avoids this failure mode upstream by using continuation seeding (`书名：《` at the end of the prompt) instead of `书名：[XXX]`, but the Checker still guards against it.

## Rule 8 — Golden-finger embodiment

The whole point of the user supplying a `golden_finger` field is that the protagonist's unique mechanic should drive the story. A common LoRA failure is to mention the golden finger in the outline but ignore it in the chapter prose. Rule 8 requires at least one 2-4-character keyword from the `golden_finger` string to appear somewhere in the chapter:

```python
gf_keywords = re.findall(r'[一-鿿]{2,4}', setup['golden_finger'])
# filter generic stopwords ('一块', '神秘', '可以' etc.)
if not any(k in text for k in gf_keywords):
    fail("golden-finger embodiment: none of " + keywords + " appears")
```

The filter list (`一块/神秘/可以/某个/...`) is intentionally short; if your `golden_finger` contains only generic words after filtering, the rule passes vacuously and you should write a more specific golden-finger string.

## Scoring

Each rule fail subtracts 12 points from a base of 100. A clean chapter scores 100; a chapter with three failures scores 64; eight failures scores 4. The pipeline keeps the highest-scoring attempt across up to three retries.

## What's not in the Checker

Some checks we considered but did not implement:

- **Entity traceability against the source novels.** A version that greps every generated proper noun against the original text would catch invented power tiers and made-up factions. We didn't include it because (a) `grep` requires the source novels at inference time, which we don't ship, and (b) the regex rules above already catch the most common failure modes.
- **Semantic coherence.** A meta-checker using a different LLM to read and judge the chapter would catch issues like internally inconsistent power-level claims. This is on the roadmap but not in the current version.
- **Style classifier.** A trained classifier distinguishing "web-novel" from "AI prose" would generalise beyond the format-ratio heuristic. Not pursued; the heuristic works in practice.

## Tuning for a different universe

The five-rule structure (anti-pollution, anti-AI-style, anti-regression, format, length) generalises. To adapt to a different fictional universe:

- Rule 1: update `XUEYING_WORDS` to terms from universes you want to exclude.
- Rule 3: change the canonical-protagonist name from `罗峰` to whichever character your audience wants to avoid.
- Rule 4: replace the full-width double-space heuristic with whatever paragraph convention your target style uses (e.g., for English fic, a paragraph break detector).
- Rules 2 and 5 are universe-agnostic.

The Setup, Outline, and Writer prompts (see `docs/prompts.md`) will need adapting too.

## Retry strategy

When a rule fails, the failure reason is appended to the next attempt's prompt:

```
上次写作有以下问题，本次必须避免：
- 雪鹰跨源词「开辟境」出现 2 次（绝对禁止）
- AI 短句分行（连续 3+ 行 8 字以内句号结尾），位置 1247
```

Empirically, natural-language feedback to the model works better than structured re-prompting. The model treats the failure message as in-context instruction.

Each retry also raises the sampling temperature by 0.05 (0.85 → 0.90 → 0.95) and uses a different random seed, to escape local minima.
