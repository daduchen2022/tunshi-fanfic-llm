# Tunshi Star Fanfic LLM

End-to-end pipeline for generating canonical-faithful fanfiction for the Chinese sci-fi novel series 《吞噬星空》(*Swallowed Star*) by 我吃西红柿.

The system takes an optional theme and produces three full chapters of original-protagonist fanfic that respect the source universe's lore, character viewpoint rules, and prose style. An adversarial checker rewrites chapters that violate any of eight hard rules.

[![License: Research](https://img.shields.io/badge/license-Research%20only-yellow)](LICENSE)
![Python](https://img.shields.io/badge/python-3.11-blue)

---

## How it works

Three independent components compose at inference time.

### 1. Worldview RAG

The 30 MB of source text in the three original novels is far too long to ground generation by retrieval directly. Instead, the system grounds generation against a curated layer of annotations:

```
Source novels (3 books, ~30 MB)
        │
        ▼  (offline, one-time)
        Annotate with an instruction-following LLM,
        manually audited for accuracy
        │
        ▼
tier_notes.md — ~11,700 annotated entries, each with [TSK1, line N] citations
        │
        ▼
chunks_with_category.jsonl — ~11,600 chunks tagged with
  source / category / about_protagonist / raw_line
        │
        ▼  BGE-zh-base-v1.5, 768-dim embeddings
vectordb/ — local ChromaDB, cosine similarity, metadata filter
```

The original novels never enter the vector store. Only the curated annotations do. The source text is the *ground truth*; the annotations are the *searchable* layer.

A 50-record sample of the annotation schema is in [`data/tier_notes_sample.jsonl`](data/tier_notes_sample.jsonl); see [`docs/data_schema.md`](docs/data_schema.md) for fields.

### 2. Style LoRA

A LoRA adapter on top of Qwen2.5-7B specialises the base model on Chinese web-novel prose, trained on a corpus of 120 fanfiction books (~32,600 chapters). Class-weighted sampling biases the adapter toward canon-adherent works.

```
LoRA configuration:
  r = 16, alpha = 32, dropout = 0.05
  target_modules = q/k/v/o + gate/up/down projections
  bf16, gradient checkpointing, cosine LR
  2 epochs, effective batch size 32, max_len 2048
```

The adapter does not try to memorise the universe's lore — that's the RAG's job. It only learns prose cadence, vocabulary, and pacing.

### 3. Adversarial generation

```
                  ┌────────────────────────────────────────────┐
       user ────▶│ Setup    (protagonist name, time, place,    │
                  │           tier, golden finger — user-filled│
                  │           dict, not LLM-generated)         │
                  └────────────────────┬───────────────────────┘
                                       ▼
                  ┌────────────────────────────────────────────┐
                  │ Outline  (book title + 3 chapter titles +  │
                  │           4 plot beats per chapter, all     │
                  │           invented by the LoRA model)      │
                  └────────────────────┬───────────────────────┘
                                       ▼
              for each chapter:
                  for attempt in 1..3:
                    ┌──────────────────────────────────────────┐
                    │ Retriever  → top-k worldview chunks      │
                    │ Writer     → ~2500 Chinese chars         │
                    │ Checker    → 8 hard rules                │
                    │ if passed: break                         │
                    │ else: inject the failure reason into     │
                    │       the next prompt, raise temperature │
                    └──────────────────────────────────────────┘
                                       ▼
              three chapters + setup.json + check_log.json
```

The eight Checker rules look for cross-universe contamination, AI-style sentence fragmentation, protagonist drift toward the canonical lead, paragraph-format deviation, chapter length, training-corpus meta-noise residue (author asides, reader-facing solicitations), placeholder echo (the model parroting `【主角】X` template scaffolding), and golden-finger embodiment (the user-supplied protagonist mechanic must actually appear in the prose). See [`docs/checker_rules.md`](docs/checker_rules.md) for the exact implementation.

A sample of the expected output format is in [`examples/`](examples/).

---

## Quickstart

The complete pipeline runs end-to-end in a single Colab notebook: [`lora/colab_full_pipeline.ipynb`](lora/colab_full_pipeline.ipynb).

### 1. Get the pre-built artifacts

Both the LoRA adapter and the vector store are distributed via [GitHub Releases](../../releases). After downloading, upload them to your Google Drive at:

```
My Drive/
└── tunshi_lora/
    ├── checkpoints/final/      ← LoRA adapter (from tunshi_lora_v1.zip)
    └── vectordb/               ← ChromaDB store (from vectordb.zip)
```

### 2. Run the notebook

1. Open [`lora/colab_full_pipeline.ipynb`](lora/colab_full_pipeline.ipynb) in Colab.
2. Set runtime to **GPU → A100** (or L4 fallback), High-RAM enabled.
3. Run cells top-to-bottom. The notebook installs dependencies, loads the base model from Hugging Face, attaches the LoRA, connects to the vector store, then generates three chapters with the adversarial Checker.

Total wall time: ~30-50 minutes on an A100 (including the first-time download of the 15 GB base model).

### 3. Customise

The notebook contains all generation logic inline. To tune the behaviour:

- **Setup**: edit the `SETUP` dict in cell 5 (protagonist name, time/place anchors, tier, golden finger). The book title and chapter titles are *not* set here — they come out of Cell 7 below.
- **Checker rules**: edit cell 6 (e.g., add terms to the blacklist, adjust thresholds). See [`docs/checker_rules.md`](docs/checker_rules.md).
- **Prompts**: cell 7 (Outline — invents book title, chapter titles, plot beats) and cell 8 (Writer — generates each chapter) contain the prompt templates as f-strings. See [`docs/prompts.md`](docs/prompts.md) for the design rationale.

---

## Repository layout

```
README.md                          this file
LICENSE                            MIT for code, research-only for adapter
requirements.txt                   pinned versions for transformers / peft / etc.

lora/
  colab_full_pipeline.ipynb        the complete pipeline (one notebook)

docs/
  architecture.md                  why the three-component split
  checker_rules.md                 the eight Checker rules + retry strategy
  prompts.md                       Outline / Writer prompt design
  data_schema.md                   tier_notes JSONL field spec

data/
  tier_notes_sample.jsonl          50 sample annotation records

examples/
  setup.json                       example protagonist anchors
  sample_chapter.md                example chapter output (format demo)
```

---

## Data and license

This repository contains **code, design documents, a 50-record annotation sample, and one example chapter excerpt**. It does **not** contain:

- The original novels (《吞噬星空》, 《吞噬星空 2 起源大陆》, 《雪鹰领主》) — copyright 我吃西红柿 / Qidian
- The fan-written derivative works used for LoRA training — copyright their respective authors
- The full annotation corpus (~11,700 records)

The pre-trained adapter and vector store distributed via [Releases](../../releases) are for **academic and research use only** under the terms in [LICENSE](LICENSE).

---

## Citation

```bibtex
@software{tunshi_fanfic_llm,
  title = {Tunshi Star Fanfic LLM},
  year  = {2026},
  note  = {RAG + LoRA pipeline with adversarial checker for canon-faithful generation}
}
```

## Acknowledgements

Anthropic's Claude (Claude 4.7) was used during development to assist with:

- generating the worldview annotation layer from the source novels (the LLM extracts factual entries which are then manually reviewed and corrected — see the *How it works* section above);
- drafting boilerplate code, prompt templates, and documentation, all subsequently reviewed and edited by the author.

The author has reviewed and is responsible for all final content, design decisions, and outputs in this repository.
