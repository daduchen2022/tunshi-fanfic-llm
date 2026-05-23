# Data schema

This document specifies the JSON Lines schema for the worldview annotations used by the RAG. A 50-record sample is at [`data/tier_notes_sample.jsonl`](../data/tier_notes_sample.jsonl).

## Record format

One JSON object per line:

```json
{
  "chunk_id":      "tsk1_00012",
  "source":        "TSK1",
  "category":      "境界体系",
  "about_luofeng": false,
  "h2_section":    "一、境界体系",
  "h3_section":    null,
  "raw_line":      237,
  "text":          "**[TSK1, 行 237]**：怪兽军方分级 **S 级 / SS 级** ..."
}
```

## Fields

| Field | Type | Description |
|---|---|---|
| `chunk_id` | string | Unique id within the source. Convention: `<source_lowercase>_<5-digit sequence>` |
| `source` | enum | `"TSK1"` (吞噬星空) or `"TSK2"` (吞噬星空 2 起源大陆). Xueying Lord is intentionally excluded — see `docs/architecture.md` |
| `category` | enum | One of: `境界体系` (tier system), `人物` (characters), `势力组织` (factions), `地点` (locations), `事件` (events), `道具功法` (items & techniques), `概念规则` (concepts & rules), `罗峰专栏` (protagonist-specific) |
| `about_luofeng` | bool | `true` if the entry is about the canonical protagonist. Used at retrieval to filter when a fanfic wants to avoid or include protagonist-adjacent context |
| `h2_section` | string \| null | Heading hierarchy from the source annotation document, top level |
| `h3_section` | string \| null | Heading hierarchy, second level |
| `raw_line` | int | Line number in the annotation document. Useful for tracing back |
| `text` | string | The annotation itself. Must contain at least one citation anchor like `[TSK1, 行 N]` |

## Citation anchor format

Inside the `text` field, every factual claim is followed by one or more citation anchors referencing line numbers in the source novel:

```
[TSK1, 行 1859]                       single line
[TSK1, 行 1859-1861]                  range (ASCII hyphen)
[TSK1, 行 54270–54276]                range (en-dash, common in CJK documents)
[TSK1, 行 3842, 3894]                 multiple individual lines
[TSK1, 行 7398, 7401-7405]            mixed
[TSK1, 行 69138–69141/69245-69258]    multiple ranges joined by /
```

Reference regex:

```python
ANCHOR_PATTERN = re.compile(
    r"\[TSK[12],\s*行\s*\d+(?:\s*[-–—,/]\s*\d+)*\]"
)
```

## Validation rules

Records used at embedding time must satisfy:

1. Required fields present: `chunk_id`, `source`, `category`, `raw_line`, `text`.
2. `source` is `TSK1` or `TSK2`.
3. `raw_line` is a positive integer.
4. `text` is at least 10 characters long.
5. *(soft, ≤ 1 % violation rate tolerated)* `text` contains a citation anchor.

## How the records are produced

These annotations are not raw chunks of the source novels. They are written by:

1. An instruction-following LLM reading the novels and extracting factual statements about the universe.
2. Manual review for accuracy, deduplication, and consistent categorisation.
3. Adding citation anchors back to specific line ranges in the source.

The full corpus contains around 11,600 records. Only a 50-record sample is included in this repository (see `data/tier_notes_sample.jsonl`); the full corpus is research material. To build a comparable RAG on a different source, follow steps 1-3 above on your own material and embed the resulting JSONL with BGE-zh-base-v1.5 or an equivalent model.
