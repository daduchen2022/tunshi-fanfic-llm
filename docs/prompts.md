# Prompt template design

The pipeline has two prompts (Outline and Writer) plus a fixed Setup that the user supplies directly. Each is engineered for a specific failure mode.

The prompts live inline in `lora/colab_full_pipeline.ipynb`; this document explains *why* they look the way they do, so anyone tweaking them knows what each section is preventing.

---

## 1. Setup — user-supplied, not LLM-generated

The pipeline does **not** ask an LLM to generate the protagonist's anchors. The user fills a dict in Cell 5:

```python
SETUP = {
    'protagonist_name': '李夜',
    'time_anchor':      '某个纪元，地球这个时代（具体年代未明）',
    'place_anchor':     '宇宙某处（不限于地球）',
    'tier_anchor':      '恒星级武者',
    'golden_finger':    '一块神秘的紫色晶石',
}
```

Why not an LLM-generated Setup?

- The Setup is *the user's creative input*. Letting an LLM pick the protagonist name, the era, and the golden finger leaves nothing for the user to author. The whole pipeline starts to feel like a random story generator.
- Five fields are easy to fill by hand. The cost of "LLM generates Setup" (JSON parsing, retry on bad format, no user agency) is much higher than the cost of asking the user to type five strings.

An earlier version of this project did have an LLM-generated Setup with full S1-S5 thought-chain output (time anchor, place anchor, tier anchor, whitelist, blacklist, causal chain, perspective wall). It was dropped because the JSON-parsing brittleness outweighed the supposed convenience.

The Outline prompt below does still embed the spirit of S1-S5 — every plot beat must respect the user's time/place/tier anchors — but the anchors themselves come from the user's dict, not the model.

---

## 2. Outline prompt — book title, chapter titles, and plot beats

The Outline prompt is the only place the model is asked to *invent* high-level story structure. It produces the book title, the three chapter titles, and four plot beats per chapter. All later cells read these back via regex.

### Multi-query RAG

Before the prompt, the Outline cell does multi-angle retrieval against the worldview store. A single query over the protagonist and tier alone misses most of the relevant context. Cell 7 issues six queries simultaneously and merges:

```python
rag_queries = [
    f'{tier_anchor} 修炼 突破',
    f'{golden_finger}',
    f'{place_anchor}',
    f'{time_anchor} 历史',
    f'{protagonist_name} 主角',
    '武者 境界 体系',
]
hits = rag_search(rag_queries, top_k=6, total_max=24)
```

The merge de-duplicates by `(source, raw_line)` and sorts by cosine distance. This typically returns 15-20 unique chunks, vs. ~8 with a single query.

### Era-aware soft warning

If the user's Setup hints that the story is far from the canonical Luo Feng era (the time/place fields contain markers like `亿万纪元`, `远古`, `平行`, `之前`, `之后`, etc.), the prompt adds an explicit warning:

```
⚠️ 本作时空 远离 原著《吞噬星空》主线时代——请把上面的世界观参考
视为「历史背景或宇宙级常识」，不要让原著的人物（罗峰、洪、雷神、
坐山客、巴巴塔、宇宙尊者、九剑尊者等具名角色）作为本作主要人物出现。
```

Without this, the RAG returned canonical-era context pulls the model right back into Luo Feng's story.

### The prompt itself

```
请为《吞噬星空》衍生宇宙写一部 3 章原创小说大纲。

【主角设定（必须严格遵守）】
- 主角姓名：{protagonist_name}
- 起始境界：{tier_anchor}
- 时空背景：{time_anchor} · {place_anchor}
- 金手指（核心驱动）：{golden_finger}

【创作要求】
1. 三章必须围绕「金手指」展开：第1章首次激活/感知它，
   第2章用它解决一个具体冲突，第3章因它招致更大格局。
2. 章节名 2-4 个中文字，体现各章主题，不要叫《序章》《楔子》。
3. 书名 4-8 个中文字，反映「{protagonist_name}」与金手指的关系。
4. 每章 4 个情节点：场景 + 冲突 + 主角动作 + 章末小钩子。
5. 主角周围的角色全部新造，不要套用原著角色名。
{remote_clause_if_applicable}

【参考世界观（RAG 检索结果）】
{rag_context}

---

请直接输出，不要任何前言。开始：

书名：《
```

### Continuation seeding (the most important detail)

The prompt does **not** end with `书名：[填入这里]`. Earlier versions of this notebook used bracketed placeholders like `书名：[4-8 字的中文书名]`, and the model copied the brackets verbatim into its output — producing literal `《[书名]》` results.

Instead, the prompt ends with `书名：《` and the model fills in directly. Parsing then reads the first `》` to extract the book title.

```python
m_book = re.search(r'书名[:：]\s*《\s*([^》\n]+?)\s*》', outline)
m_chapters = re.findall(r'第\s*[1-3一二三]\s*章\s+([^\n]+)', outline)
# also strip any 【XX】 [XX] residue that did leak through
book_title = strip_placeholder(book_title)
chapter_titles = [strip_placeholder(t) for t in chapter_titles]
```

If parsing fails — the model omits the 书名 line — the notebook falls back to `f'{protagonist_name}传'` for the book title and `第N章` for any missing chapter title.

---

## 3. Writer prompt — per-chapter, style-anchored, with retry feedback

The Writer is called once per chapter (and again per retry). It does its own multi-query RAG retrieval per chapter, banks the canonical-character residue when the Setup says the era is remote, and explicitly demands the golden finger appear in the chapter prose.

```
{BOOK_META}
<chapter_title>第{chapter_idx}章 {chapter_title}</chapter_title>

【主角与设定（不可违背）】
- 主角：{protagonist_name}，{tier_anchor}
- 时空：{time_anchor} · {place_anchor}
- 金手指：{golden_finger}（**本章必须有具体情节体现这个金手指**）

【参考世界观（仅用作背景常识，不要照搬人物名）】
{ch_rag}

[ if era-remote, also: ]
本章绝对不能出现以下原著人物或角色名：
  罗峰、洪、雷神、坐山客、巴巴塔、九剑尊者、紫月始祖、
  宇宙至尊、宇宙尊者、元、黑沼

[ on retries only: ]
【上次写作存在以下问题，本次必须避免】
  - Rule 6 (meta-noise): leaked author/reader/serialisation markers: ['未完待续', ...]
  - Rule 8 (golden-finger embodiment): none of [...] appears in this chapter
...

【前章末尾衔接】
{prev_chapter_tail[-300:]}

【写作要求】
- 番茄式短段落（中文全角双空格缩进，对话独立成段），约 2500 中文字。
- 主角名「{protagonist_name}」至少出现 8 次。
- 本章必须围绕「金手指」展开至少一段具体描写。
- 不要出现「未完待续」「下一章」「作者的话」「求收藏」「读者评论」等同人文套话。
- 不要使用方括号或全角括号包裹的占位符（如「【主角】」「[书名]」）。

直接开始写正文，不要给前言或总结：

　　
```

Notable choices:

- **`<book_meta>` and `<chapter_title>` tags** mirror the format the LoRA was trained on. The training corpus wraps every chapter in these tags, so re-using them at inference primes the model to produce same-shape output.
- **`BOOK_META` is built from the Outline-parsed book title**, not hardcoded.
- **Per-chapter multi-query RAG** (4 queries × top_k=4 → cap at 10) gives the Writer different context than the Outline — focused on combat / golden-finger mechanics / location for that specific chapter.
- **Era-remote ban list** is the same heuristic as Cell 7 and is the main fix for the "亿万纪元前 yet 九剑尊者 appears" failure mode.
- **"本章必须有具体情节体现这个金手指"** is the explicit instruction that pairs with Checker rule 8 (golden-finger embodiment). It is repeated in the requirements block.
- **The negative list of stock fanwork phrases** prevents Rule 6 failures upstream rather than only catching them in the Checker.
- **`前章末尾衔接` (last 300 chars of previous chapter)** is what gives the three chapters narrative continuity. Without it, each chapter is a standalone.

The trailing `　　` (full-width double-space) is the start of the first paragraph — letting the model continue from inside an already-formatted paragraph rather than asking it to produce the header itself.

---

## Retry feedback

When the Checker fails a chapter, the failure reasons are injected verbatim into the next attempt:

```
上次写作有以下问题，本次必须避免：
- 雪鹰跨源词「开辟境」出现 2 次（绝对禁止）
- AI 短句分行（连续 3+ 行 8 字以内句号结尾），位置 1247
```

We tried two more structured alternatives:
- "Set `forbidden_terms = ['开辟境', ...]`" — model treated it as data, not as a constraint.
- "Output JSON with `avoided_terms` field" — broke the model out of narrative mode.

Plain Chinese instruction works best: the model treats the failure description as in-context guidance and naturally avoids the offending pattern.

Temperature is bumped by +0.05 on each retry (0.85 → 0.90 → 0.95) and the seed changes, so a retry is not just rerunning with the same dice.

---

## Tuning parameters

| Parameter | Default | Why |
|---|---|---|
| temperature (initial) | 0.85 | balance creativity vs determinism |
| temperature (retry +) | +0.05 per retry | escape local minima |
| top_p | 0.9 | standard for creative generation |
| repetition_penalty | 1.1 | light — too high hurts paragraph cohesion |
| max_new_tokens (Outline) | 1200 | enough for 3 chapter outlines |
| max_new_tokens (Writer) | 1800 | targets ~2500 Chinese chars |
| RAG top_k | 6-10 | diminishing returns above 10 |
| RAG distance threshold | 0.55 | tighter misses canonical references; looser adds noise |
| Outline RAG context size | ~1000 chars | enough for variety without blowing context |
| Writer RAG context size | 5 chunks ~750 chars | per-chapter focused |

---

## Adapting to a different universe

The Outline and Writer prompts are templates with universe-specific content inside generic structure. To port to a different fictional universe:

1. **Outline prompt**: change `《吞噬星空》` to your universe name, adjust the example chapter-name lengths if your style differs.
2. **Writer prompt**: change `番茄式` style descriptor to whatever your target style is. For English fanfic, replace `中文全角双空格` with `indent each paragraph with two spaces` or whatever convention applies.
3. **`BOOK_META` format**: keep the tag structure (the LoRA learned to predict from it), but change the category labels.

The Checker (`docs/checker_rules.md`) needs more thorough adaptation — see that document.
