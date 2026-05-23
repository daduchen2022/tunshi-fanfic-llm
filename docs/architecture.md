# Architecture decisions

This document explains *why* the system has the shape it has. For the *how*, see `README.md` and the code.

## Why split worldview from style

A reasonable first attempt would be: take the original novels, fine-tune the LLM on them, and let the model produce fanfic. We explicitly didn't do this.

Three reasons:

1. **A 30 MB corpus is small for pre-training a 7B model.** The model would memorise rather than learn structure, producing paraphrases of canonical chapters.
2. **The lore checks we want are sharper than what fine-tuning gives.** "Did the model invent a power tier that doesn't exist?" is hard to enforce by training. It is easy to enforce by retrieval + a regex sweep over named entities.
3. **The fanfic style we want is distinct from the source author's voice.** The fanworks corpus (210 MB across 120 books) is in a recognisable web-novel cadence shared by many authors — short paragraphs, dialogue-driven action, concrete numerics. That style is what the LoRA should learn. The lore is what the RAG should provide.

So: the LoRA is trained on **fanworks**, not the source novels. The RAG indexes **curated annotations of the source novels**, not the novels themselves.

## Why annotations instead of the raw novels in the RAG

Three choices for what to embed:

- **Raw novel chunks (200-token spans).** Easy. But action scenes, recap, and dialogue all chunk identically, and the retrieved chunks are usually irrelevant to "what is the standard power tier for X?"
- **Question-answer pairs.** Tried in early prototypes. Brittle: a question phrased differently from the QA pair won't match.
- **Curated structured annotations.** What we do. An LLM (Claude 4.7 in our case) processed the novels into ~11,700 short factual entries, each tagged by category (`境界体系`, `人物`, `道具功法`, `概念规则`, …) and carrying a `[TSK1, 行 N]` line citation back to the source.

The annotations are the *searchable* layer; the raw novels are the *ground truth* layer accessed via `grep` when a generated entity needs verification (see `rag/04_grep_original.py`).

This means an entity hit in the RAG is a high-signal match: it is by construction a factual claim about the canon, not just a phrase that happened to appear in dialogue.

## Why continued pre-training, not instruction tuning, for the LoRA

Web novels are long-form continuous prose. The information that defines the style is in things like:

- when paragraphs break
- where dialogue interleaves with description
- how power-level comparisons are phrased
- pacing across chapter boundaries

If we tokenise everything as Q&A pairs, that structure is destroyed. Each example becomes a short context-answer, the cross-chapter flow disappears, and the model learns to answer questions about the universe instead of writing in it.

Continued pre-training preserves the structure. Each training example is a full chapter wrapped in `<book_meta>` and `<chapter_title>` tags with the EOS sentinel at the end. The model sees the chapter as a continuous prediction task, like the original pretraining.

## Why class-weighted sampling

The 120-book corpus is uneven:

- A few books are strict canon-adherents that stay inside TSK2's setting (label `D1`).
- More are looser fanfic that bend the universe (label `A`, the bulk of the corpus).
- A handful are off-universe or genre-mixing (`B`, `C`, `D2`).

Uniform sampling would let `A` dominate and the model would drift away from canon-adherence. Per-class weights at training time (`D1` 2.0×, `A` 1.0×, `B`/`C`/`D2` 0.3×–0.5×) keep `D1` over-represented in gradients despite being a small fraction of the data. See `lora/01_pack_pretrain.py`.

A small auxiliary corpus from a different novel (Xueying Lord) is included at very low weight (0.3×) to slightly broaden style without contaminating universe-specific terminology — the checker catches anything that leaks through.

## Why an adversarial checker instead of soft regularisation

Many fine-tuning approaches try to bake the "don't say X" constraint into training (e.g., DPO with negative examples). For this project that's the wrong tool:

- **The forbidden set is small and well-defined.** A regex over ~30 cross-universe terms is more reliable than a learned dispreference signal.
- **Failure rates are low (<10 %).** Cost of training to drive them lower exceeds cost of just rejecting at inference time.
- **Auditable rules > learned behaviour.** A regex-based checker is debuggable; a learned constraint isn't.
- **No retraining cost to add a rule.** Updating the rule set is editing a Python file.

So the checker runs after generation, on each candidate chapter. It scores the chapter against eight hard rules, and if any rule fails, the failure reason is concatenated into the next prompt and the chapter is regenerated at a higher temperature. Up to three attempts per chapter; if all three fail, the highest-scoring attempt is kept.

The eight rules cover both intrinsic style failures (AI-style fragmentation, missing paragraph indentation, short chapters) and corpus-leakage failures (training-data residue like author asides and reader-facing solicitations, cross-universe contamination, placeholder echo of prompt scaffolding). The newest rule, golden-finger embodiment, requires the user-supplied protagonist mechanic to actually appear in the chapter prose, because the most reliable failure mode of LoRA models on this corpus is to mention a unique mechanic in the outline and then ignore it in the chapter itself.

See `docs/checker_rules.md` for the rules.

## Why no GraphRAG / HyDE / reranking

We tried plain top-k BGE-zh retrieval against the annotated chunks and got a >70 % entity-traceability rate on generated chapters (per the checker's final rule). Adding HyDE-style query rewriting did not move the metric. RAPTOR-style multi-level summarisation made retrieval slower without helping recall. A reranker bought ~3 percentage points at significantly more inference cost.

For corpora of ~10k chunks against a well-tagged annotation schema, plain vector search is sufficient. The tag-based filters (source, category, about_protagonist) do most of the work that fancier retrieval would have done.

## Why Qwen2.5-7B as the base model

Constraints:

- Chinese-native (the corpus is entirely Chinese)
- Permissive license for research use
- Small enough to train LoRA on a single A100 80 GB
- Small enough to serve inference on consumer-grade hardware via 4-bit quantisation if needed

Qwen2.5-7B fit all four; Qwen2.5-14B was too tight on the A100 in bf16; Qwen2.5-1.5B underfit and lost the cadence.

## What we'd change with more compute

- **Larger base model.** Qwen2.5-14B in bf16 would need an A100 80 GB at QLoRA, or two A100 80 GBs at full LoRA. Style fidelity would likely improve.
- **More training data.** The fanworks corpus has ~80 M tokens after weighting. Doubling that is straightforward (more crawled fanworks) and would mostly help with rare scenarios — combat scenes vs daily-life scenes, for instance, are unevenly represented.
- **An LLM-as-judge meta-checker.** The five-rule checker catches regex-detectable issues. A meta-checker (Claude or GPT-4 reading the chapter) would catch semantic issues, e.g., "this character's power level is internally inconsistent across chapters". This was scoped out of the current version because the regex rules already moved generation quality past the bar we set for ourselves.
