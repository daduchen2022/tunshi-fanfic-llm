# Sample chapter output

This is a short illustrative excerpt showing the expected format and style of a generated chapter. It is **not** a complete chapter (the pipeline produces ~2500 characters per chapter); it shows the first few paragraphs so contributors can understand what the Checker's format rules look for.

Generated using the setup in [`setup.json`](setup.json) — protagonist `李夜`, stellar-tier, with a mysterious purple crystal as the source of power.

---

## 第1章 觉醒

　　李夜睁开眼。

　　头顶是漆黑的虚空，远处几颗恒星散发着冷冽的光。他的身体悬浮在一片碎裂的星岩之间，胸口隐隐发烫——那块紫色晶石此刻紧贴着他的心脏，传来一阵阵清晰的搏动。

　　"这是哪儿？"

　　他低声自语，声音在真空中并未传出，但他自己听得见。恒星级武者的神识可以脱离声音的限制，直接在意识层面交流——这是他在突破到这个层次后才掌握的本能。

　　李夜抬起手，指尖凝聚出一团淡蓝色的星辉。这是他的本命法则——星辰本源，能将远处恒星的能量牵引到自身周围。他试着催动晶石，紫色光芒立刻顺着血脉流转，像一条小溪汇入大海。

　　力量在他体内疯狂增长。

　　不是缓慢的吸收，而是像被打开了某个阀门，恒星能量顺着晶石的纹路涌入，速度比他自己修炼快了整整一百倍。

　　"这块石头到底是什么？"

　　李夜眉头微皱。来历他不清楚，只记得自己在祖父留下的一个旧匣子里发现了它。当时他只是个初级战士，恒星级是十年后才达到的境界——但那块晶石仿佛就是为他准备的，每次突破前都会主动散发出温热。

　　远处，一道身影飞速接近。

---

This excerpt demonstrates how the eight Checker rules apply (see [`docs/checker_rules.md`](../docs/checker_rules.md) for the exact regex and thresholds):

- **Rule 1 (cross-universe contamination)**: No Xueying Lord terms (`开辟境`, `东伯雪鹰`, etc.)
- **Rule 2 (AI-style fragmentation)**: Sentences vary in length; no three consecutive short period-terminated lines
- **Rule 3 (protagonist independence)**: The protagonist `李夜` appears 6+ times in this excerpt; the canonical lead `罗峰` does not appear at all
- **Rule 4 (paragraph format)**: Every paragraph begins with full-width double-space (`　　`)
- **Rule 5 (length)**: Excerpt is ~600 chars; a complete chapter would target ~2500
- **Rule 6 (meta-noise residue)**: No author asides (`作者的话`, `求收藏`), serialisation markers (`未完待续`), or reader-facing solicitations
- **Rule 7 (placeholder echo)**: No template scaffolding like `【主角】` or `[书名]` leaked into the prose
- **Rule 8 (golden-finger embodiment)**: The protagonist's setup mechanic (a mysterious purple crystal) is concretely depicted — "胸口隐隐发烫", "紫色光芒立刻顺着血脉流转"

A complete generation produces three chapters of this format plus `setup.json` (the protagonist anchors), `outline.txt` (the per-chapter plot beats), and `check_log.json` (the Checker's per-attempt scoring trace).
