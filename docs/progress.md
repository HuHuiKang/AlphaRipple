# 进度记录：PDF → MinerU → 结构化抽取 最小验证

最后更新：2026-11-01

## 路线

| 步骤 | 内容 | 状态 |
|---|---|---|
| 0 | 环境搭建（MinerU + GPU） | ✅ 完成 |
| 1 | 准备样本 PDF | ✅ 完成（研报、半年报、招股书各 1 份） |
| 2 | MinerU 解析 + 人工评估 | ✅ 基本完成（招股书 188–190 页未测） |
| 3 | 从解析结果中定位产业链相关片段 | ⏭ 下一步 |
| 4 | Pydantic 定义关系 schema | 待做 |
| 5 | LLM 结构化抽取 + 证据校验 | 待做 |
| 6 | 小金标集评测（准确率 / 召回率） | 待做 |

## 环境

两个环境分开，互不干扰：

| 环境 | 位置 | 用途 | 怎么用 |
|---|---|---|---|
| MinerU | `..\MinerU\.venv`（Python 3.12，torch 2.14.1+cu126，mineru 4.1.3） | 只做 PDF 解析 | 先激活：`..\MinerU\.venv\Scripts\Activate.ps1` |
| AlphaRipple | `.venv`（Python 3.12，uv 管理） | 项目代码 | 不用激活，用 `uv run ...` |

- uv 装在 conda base 里，所以要在 `(base)` 下才能用 `uv` 命令。
- MinerU 的模型在 `C:\Users\hk929\.mineru\models`，basic 和 standard 两档都已经下载。
- 用 `mineru-kit parse` 在本地解析，**不加 `--remote` 就不会上传文件**。
- 跑完 AlphaRipple 的代码后，如果还要再用 MinerU，要先激活 MinerU 的环境。

## 第 2 步结论（MinerU 评估）

| 档位 | 研报 5 页耗时 | 峰值 | 图表数据 | 小表格 |
|---|---|---|---|---|
| basic | 6.3s | — | 空 | 列错位 |
| standard | 9.1s | 76% | 空 | 正确 |
| advanced | 17.1s | 100% | 转成了表格（柱状图数值是估算的，带 `~`） | 拆成了文字 |

- **年报、半年报、招股书用 standard**。半年报第 174–176 页的跨页关联交易表在 md 里合并成了一张表，4 个合计数全部验算一致。
- **研报中有图表的页面用 advanced**。图表数据只能当辅助证据，不能当精确数值用。
- 已知问题：
  - 跨页时段落会断开
  - 个别单元格里文字顺序错乱（半年报 md 第 64 行"合股"肥)海恒控股…"）
  - 半年报里没有"前五名客户/供应商"表，这张表要去年报或招股书里找

## 解析命令模板

在 AlphaRipple 目录下、MinerU 环境已激活时执行：

```powershell
mineru-kit parse "data\bronze\samples\<文件>.pdf" -o "data\silver\<输出目录>" --tier standard -f middle_json -p <页码>
```

- `-p` 的页码从 1 开始，两端都包含，可以用逗号组合，比如 `188-190,253`。
- `-f` 可选 `middle_json`（给程序处理）或 `markdown`（给人看）。
- 文件路径含有中文和全角冒号，要加引号。

## 下次从这里开始

1. **补跑缺的 JSON**（第 3 步需要页码，md 里没有页码）：
   ```powershell
   ..\MinerU\.venv\Scripts\Activate.ps1
   mineru-kit parse "data\bronze\samples\长鑫科技：2026年半年度报告.pdf" -o "data\silver\half_standard" --tier standard -f middle_json -p 174-176
   mineru-kit parse "data\bronze\samples\长鑫科技：长鑫科技首次公开发行股票并在科创板上市招股说明书.pdf" -o "data\silver\ipo_standard" --tier standard -f middle_json -p 188-190,253
   ```
   顺便用 `-f markdown` 解析招股书这几页，人工检查前五大客户表和前五大供应商表。

2. **3.1 先确认**：半年报 JSON 里，第 174–176 页的关联交易表是一个 table 块，还是多个？如果是多个，后面几块有没有表头？这个结果决定脚本要不要自己合并跨页表格。

3. **3.2–3.4 写 `src/alpharipple/locate.py`**：
   - 输入 middle_json，输出 `data/silver/chunks/<doc_id>.jsonl`。
   - 每个片段的字段：`doc_id`、`doc_type`、`page`（从 1 开始）、`block_type`、`section`、`caption`、`content`、`bbox`、`is_candidate`。
   - 处理规则：
     - 丢弃页眉、页脚、页码和"√适用 □不适用"这类勾选框文字
     - 遇到 `paragraph_title` 时更新当前章节路径
     - 表格没有 caption 时，往前找最近的 text 块当标题，并记下"单位"
     - 命中关键词的片段标为候选。关键词：前五大客户、前五名客户、前五大供应商、关联交易、主要原材料、营业成本构成、采购、销售
   - 运行方式：`uv run python -m alpharipple.locate <json路径>`

4. **验收**：
   - 半年报第 174–176 页的两张关联交易表都被标为候选，页码、标题、章节都正确。
   - 招股书第 188–190 页的前五大客户表和前五大供应商表都被标为候选。
   - 脚本最后打印总片段数、候选片段数，以及各类型片段的数量。
