---
name: paper-data-field-extractor
description: Use when a paper must be converted into a target field table with values, units, conditions and source paragraphs.
---

# 论文目标字段与数值抽取

## 用途与适用条件

按给定的目标字段表，把论文正文、表格、图注与补充材料还原成同一份字段表：每个 `field_id` 给出值、单位、统计量、实验条件、比较符与段落出处，并区分已报告、未报告、来源冲突与抽取失败。

适用：`/input/case.json`（或 `--case` 指定），可带 `/input/task.md` 与 `/input/target-fields.json`，以及论文段落或正文。输入含 `target_fields`/`fields`/`extraction_fields` 与 `paragraphs`，通常要求保留 `paragraph_id` 与逐字引文。

不适用：不根据领域常识补齐未报告字段；不把不同实验条件的数值合并为单一值；不改写、换算或四舍五入原文数值。

## 输入

- 主输入 `/input/case.json`：只读取其中真实的段落、表格与字段表，不根据文件名、标题或领域标签补造内容。
- 字段表 `target-fields.json`（或 `case.json` 内的 `target_fields`/`fields`）：每个 `field_id`、名称、别名、说明与期望单位都必须逐条处理。
- 配套材料从 `/input` 与案例文件同目录读取，读取结果记入 `result.field_list_refs` 与 `stats`，失败记入 `errors`。
- 路径解析顺序：`/input` → 案例目录 → 当前目录 → `/app`；找不到的路径写入 `errors`/`uncertainty`，不静默替换。
- 支持 TXT/MD/CSV/TSV/JSON/BibTeX/RIS/XML/HTML、DOCX/PPTX/XLSX（解包 XML）与 PDF（最小流解析）；扫描件或无文本层只登记状态。
- 仅用 CPython 3.9+ 标准库；不联网、不读取环境变量中的令牌、不修改 `/input`，只写 `--out` 与 `--work`。

## 处理流程

1. **解析字段表**：读取字段 id、名称、别名、说明与期望单位；字段表缺失时输出 `insufficient_input`。
2. **字段召回**：用字段名与别名逐句匹配段落，记录全部候选句、来源与定位。
3. **数值解析**：抽取数字、科学计数法、范围值、单位与比较符（`<`、`>`、`≤`、`≥`、约）；保留原始数值字符串。
4. **条件与统计量**：把数值所属的样本、实验设置、时间点、比较基线与统计量（误差、置信区间、p 值等）绑定到同一字段，不脱离语境单独成行。
5. **单位匹配**：只有单位与字段期望单位一致（或可明确换算并同时保留原始串）的数值才算该字段的候选；同句出现多个不同单位的数值时取匹配项，其余写入 `uncertainty`，不得直接判成 `source_conflict`。
6. **冲突判定**：不同段落给出不同数值或单位时标 `source_conflict`，保留全部候选与两处定位，不平均、不择一。
7. **未报告/失败**：有匹配句但原文说明未报告时标 `not_reported`；有文本但无法解析数值时标 `extraction_failed`。
8. **证据与语义核对**：每个报告值绑定 `paragraph_id`、逐字引文与定位；逐字匹配不等于语义正确，须核对该句是否真的在报告该字段。

## 输出契约

写入 `/app/output/result.json`（UTF-8 JSON，确定性排序，无随机、无时间戳）。顶层必须保留：

- `case_id`：与输入完全一致（缺失时为 `null`）；
- 输入 `output_contract` 声明的每一个字段名，原样保留；无法确定性映射的写 `null`，并在 `contract_notes` 说明原因；
- `status`：`success` / `partial` / `insufficient_input` / `failed`；
- `result`、`evidence`、`uncertainty`、`errors`、`stats`、`contract_notes`、`skill`、`challenge_id`。

主要交付段与必填子字段：

| 段 | 必填内容 |
|---|---|
| `result.fields[]` | 每个请求的 `field_id` 一条：`field_id`、`field`、`description`、`expected_unit`、`status`（`reported`/`not_reported`/`source_conflict`/`extraction_failed`）、`value`、`unit`、`statistic`、`condition`、`comparator`、`paragraph_id`、`quote`、`matched_sentences`、`source_ids` |
| `result.conflicts[]` | `field_id`、`field`、`kind`、`values[]`（每组 `value`、`unit`、`paragraph_id`、`quote`）、`policy` |
| `result.coverage` | `target_fields`、`reported`、`not_reported`、`source_conflict`、`extraction_failed`、`paragraph_units` |
| `result.field_list_refs` | 实际读取的字段表文件名 |
| `evidence[]` | 每条含 `field_id`、`paragraph_id`、`quote`、`locator`（页码/表行列/图注编号） |
| `needs_human_confirmation[]` | `field_id`、`locator`、`reason`：字段歧义、单位缺失、引文与字段不对应、跨页表格未还原时登记 |

字段名以实际 `output_contract` 为准；不删除、不改名、不放宽契约字段。

## 交付成品要求

1. **交付成品而不是流程说明**：脚本只产出抽取骨架；调用 Agent 必须补齐 `statistic`、`condition` 与语义核对结论，禁止 `pending`、`TODO` 或空交付段。
2. **覆盖全部字段与来源**：`target-fields.json` 里每个 `field_id` 一条，正文、表格、图注、补充材料按同一字段表处理，不得只处理首条或首段。
3. **不改写原文**：数值、单位、误差、统计量与条件限定保持字面一致，不合并、不换算、不张冠李戴；未报告字段保持 `null`，不以常识补齐。
4. **自检**：`case_id` 与输入一致；`output_contract` 每字段存在或 `null`+`contract_notes`；`status` 取上述枚举；`coverage` 计数与 `fields` 一致；重复运行结果相同。
5. **参数等价**：`--out` 与 `--output` 等价；默认 `/input/case.json` → `/app/output/result.json`。

## 异常与降级

| 情况 | 处理 | 输出表现 |
|---|---|---|
| `case.json` 缺失或非法 JSON | 不改写输入，写出结构化失败结果 | `status: insufficient_input`（解析错误为 `failed`），`errors` 记原因 |
| 字段表缺失 | 不猜测字段清单 | `status: insufficient_input`，`errors` 说明 |
| 扫描件无文本层、文件损坏或无权限 | 登记状态并跳过，继续其余材料 | `read_status`/`uncertainty` 标注 |
| 字段原文未报告 | 保留 `null`，不推断 | `status: not_reported`，`uncertainty` 说明 |
| 有文本但无法解析数值 | 不猜数值 | `status: extraction_failed`，`uncertainty` 说明 |
| 多条来源数值冲突 | 保留全部候选与定位，不平均不择一 | `status: source_conflict`，`conflicts` 并列两组 |
| 运行异常 | 捕获后输出结构化失败 | `status: failed`，`errors` 含异常类型 |

进程正常结束（含失败分支）退出码恒为 `0`，失败类别只看 `status`，不要用退出码判断成败。

## 限制

只解析显式数字与单位；图表数值、公式推导值与跨段指代可能漏检。字段别名覆盖范围取决于输入字段表。

## 最小调用示例（示例，非官方案例结果）

```bash
python scripts/run_skill.py --case /input/case.json --out /app/output/result.json --work /app/work
```

示意输入（合成，仅演示接口）：

```json
{"case_id":"synthetic-smoke","output_contract":{"fields":"array"},"target_fields":[{"field_id":"F1","name":"yield","unit":"%"}],"paragraphs":[{"paragraph_id":"p1","text":"The yield was 82.5 % under nitrogen."}]}
```

示意输出（合成，仅演示字段形状，不代表任何真实案例或评分）：

```json
{"case_id":"synthetic-smoke","fields":[{"field_id":"F1","field":"yield","status":"reported","value":82.5,"unit":"%","paragraph_id":"p1"}],"status":"success","result":{},"evidence":[],"uncertainty":[],"errors":[],"stats":{},"contract_notes":{}}
```

## 自查清单

- [ ] `case_id` 与输入一致；`output_contract` 每个字段都存在，或记为 `null` 并写入 `contract_notes`。
- [ ] 每个字段值与逐字引文、段落定位一致，未编造数值、单位、作者或页码。
- [ ] 未报告、抽取失败、来源冲突与未读材料均已分别登记，未写成“全部完成”。
- [ ] 重复运行结果相同；未联网、未修改 `/input`；未声称平台评分或官方案例结果。
