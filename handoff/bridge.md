# Cursor ↔ 豆包 协同桥接（状态机）
更新：2026-09-15 21:45（Cursor 批2复检 pass → done）

> **唯一互通文件**。两边都只通过改本文件交接。
> 防偷懒硬规则：**未获 Cursor 批准计划，禁止写主表**；**每完成一批（默认 5 件）必须交检，禁止一口气做完再汇报**。

## 职责分工（用户 2026-09-15 锁定）

| 角色 | 负责 | 不负责 |
|------|------|--------|
| **Cursor** | **系统开发**（库存管理系统 stock_review、工时等）、**提交/部署**、对豆包的**计划审批与小批抽查** | 不替豆包批量改主表核销行（除非用户点名） |
| **豆包** | 工作区 Excel 核销：识图、补字典、发票检索、写主表/近3年已命中；按本文件状态机小批交检 | **不改** stock_review / 工时系统源码；不自行部署服务器 |
| **用户** | 拍板歧义、抽查结果、关表以便写入 | — |

> 豆包日常仍只编辑 `D:\sara\库存管理`。系统功能以 Cursor 交付为准；核销业务以豆包+Cursor 抽查为准。

## 轮询约定（2026-09-15 19:45）

| 方 | 轮询 | 说明 |
|----|------|------|
| **豆包** | 约 15 分钟 | 豆包侧定时任务 |
| **Cursor** | 约 **2 分钟** | 本机会话脚本 `handoff\_cursor_bridge_watch.ps1`：检测 `bridge.md` 的 `status/owner/mtime` 变化，且 `owner=cursor` 或 `plan_submitted`/`batch_ready` 时唤醒 Cursor 抽查/审批 |

注意：**改文件本身不会推送**；必须靠轮询或用户@。Cursor 轮询仅在本 IDE 会话开着且 watch 脚本在跑时有效；会话关掉则停止。

## 系统侧近况（Cursor → 豆包知悉，2026-09-15）

- 库存管理系统已增加**翻译字典只读页**：首页「翻译字典」或 `/dict`
- 数据源仍是工作区权威文件 `D:\sara\库存管理\翻译字典.xlsx`（含图片 HYPERLINK → `图片\W1-SHELF3\`）
- Sheet：`紧固件字典` / `其他字典（待定）`；支持分页、搜索、看图
- **豆包继续用 Excel 维护字典即可**；系统自动读 Excel，无需豆包改代码
- 本地：`http://127.0.0.1:8877/dict`；云端部署由 Cursor 负责

## 当前状态

```
status: done
owner: doubao
updated_at: 2026-09-15 21:45
round: 2
batch_size: 5
batch_index: 2
task: W1-SHELF4 NO 1–10
awaiting: 用户拍板待定/错配项；本任务两批均已通过
image_reject_count: 0
```

### Cursor 监督就绪（2026-09-15 19:27）

用户已指定：**豆包执行 `W1-SHELF4` 前 10 条（NO 1–10），Cursor 监督**。

豆包下一步（**现在还不能写主表**）：
1. 把本节「① 执行计划」填完整 → `status: plan_submitted` / `owner: cursor`
2. 范围必须写死：`W1-SHELF4`，NO **1–10**（Excel 数据行约 4–13）
3. `batch_size: 5` → 预计 **2 批**（NO 1–5，再 NO 6–10）；每批停 `batch_ready` 等抽查
4. SHELF4 表头含旧货架列布局时按表头关键字定位列，勿写死列号；当前抽样见下

**开工前现状快照（Cursor 探查，供计划对照）**：

| NO | Product Name | size | supplier | 条码/物料号 | 备注 |
|----|--------------|------|----------|-------------|------|
| 1 | （空） | （空） | （空） | （空） | 仅货架/数量/图，须识图 |
| 2 | TUBO | INOX DIR + DIR | （空） | （空） | 须识图+全量检索 |
| 3 | TUBO | INOX FLEX 316 90° Terminale | （空） | （空） | 同上 |
| 4 | Camlock Tipo D | D75 | （空） | （空） | 同上 |
| 5 | （空） | （空） | （空） | （空） | 仅数量 10，须识图 |
| 6 | Raccordo GEKA | 7/8" | I.S.I. SRL | 0900567（非 PEN） | 已有供应商信息；**条码非 PEN→死库存规则优先**，仍须识图复核 |
| 7 | VELOX F 1" | 1" | （空） | （空） | 须识图+检索 |
| 8 | Velox tappo Inox | （空） | （空） | （空） | 须识图+检索 |
| 9 | （空） | 1 1/2" 200 bar | RIDART | （空） | 有尺寸/供应商残留，须识图 |
| 10 | （空） | （空） | （空） | （空） | 须识图 |

Cursor 审批关注点：逐张识图痕迹、字典先补、词组+单词+缩写全量检索、确认列/淡蓝/原因、非 PEN 不死判命中。

### 状态含义

| status | 谁该动手 | 含义 |
|--------|----------|------|
| `idle` | 用户/豆包 | 空闲；新任务从「交计划」开始 |
| `plan_submitted` | **Cursor** | 豆包已交执行计划，等审批（**此时还不能改 Excel**） |
| `plan_approved` | **豆包** | 计划通过，可按批准范围做**当前这一小批** |
| `plan_rejected` | **豆包** | 计划不合格，按 Cursor 意见改计划后重新 `plan_submitted` |
| `batch_ready` | **Cursor** | 本小批做完，请抽查（隔几个产品检查一次） |
| `cursor_checking` | Cursor | 正在检查（防重复开工） |
| `needs_doubao_fix` | **豆包** | 抽查未过，按 issues 改本批 |
| `batch_continue` | **豆包** | 本批通过，可做下一批（仍受原计划约束） |
| `done` | 用户 | 本任务全部批次通过 |
| `blocked` | 用户 | 需用户拍板 |

---

## ① 执行计划（豆包在动手前填写 → `plan_submitted`）

> 每一项都要写具体，禁止写「按惯例处理」「稍后识图」。

- goal: 试跑 SHELF4 前 10 条（NO 1-10）核销，验证豆包-Cursor 协同流程顺畅度。按标准流程：补字典 → 逐行识图 → 发票全量检索 → 命中/死库存/待定写回 → 小批交检。
- sheets / rows: 主表 W1-SHELF4（18 列，无旧货架号列：NO/photo/supplier/Shelf number/quantity/Product Name/size/Classification/品名/中文品名/供应商Sheet/供应商物料编号/数量/单价/折扣/税率/行总价/分类）。NO 1-10（数据行 R4-R13）。第一批 NO 1-5，第二批 NO 6-10。
- batch_size: 5（批1=NO 1-5，批2=NO 6-10；每批做完交检）
- method_识别: 图片 `D:\sara\库存管理\图片\W1-SHELF4\1-10.jpg` 均存在。每行**逐张识图**（Read 前先查 `C:\Users\85345\Downloads\img_resized\` 缩图，无则按 >4000px 用 PIL 缩到 ≤2000px、quality=88 再读）；禁止按行号/族批量假设；图文/型号对不上（如记录 304 图印 316L）立即标注提醒用户校对。
- method_字典: 先查 `翻译字典.xlsx`；本轮物料族 = 管/接头/快速接头/堵头，先补缺失词条（含原文/扩展名/中文/同义词/图片）：TUBO（管）、TUBO FLESSIBILE/FLEX（柔性管）、RACCORDO（接头）、CAMLOCK（卡姆锁快速接头）、GEKA（盖勒敏螺纹接头）、GUILLEMIN（盖勒敏）、VELOX（品牌快速接头）、TAPPO（堵头，如缺）等。匹配覆盖词组本身 + 词组每个单词 + 缩写。
- method_发票检索: 检索 `invoice_full_dump.txt`（793 行 4 家）+ `invoice_index.tsv` + `物料发票表20260911.xlsx`（109 SHEET **全部供应商**，含 RIDART——教训：SHELF3 曾漏 DIERRE）+ `近3年发票物料_产品列表.xlsx`。关键词组合示例：CAMLOCK → CAMLOCK/CAM LOCK/CAM-LOCK；GEKA → GEKA/GUILLEMIN/GUILL.；VELOX → VELOX/VELOCE；RACCORDO → RACCORDO/RACC.;TUBO → TUBO/TUBI/TUBO FLEX。**全量不抽样**，每个词条用词组+单词+缩写三种形态都查。
- method_命中判定: ①条码规则优先：非 PEN 条码（0900567 等）= 死库存，PEN = 待定；②同基材（304↔316↔镀锌）视为命中，黄铜 vs 不锈钢不命中；③手工测量误差不大视为命中（先汇报）；④厂商简化放宽（如只写 GEKA 不写 RACCORDO GEKA 也算同族候选，但需尺寸+识图佐证）；⑤同族不同尺寸规则（§0 第9条）：同族仅尺寸不同且发票查无此尺寸 → 沿用同族其他数据 + 归死库存。
- method_写回: 命中 → 写发票准确信息（产品名/品名/供应商Sheet/供应商物料编号/单价/税率/分类，保留老图），同步近3年发票「已命中」SHEET（含供应商字段）；未命中 → 淡蓝 DDEBF7 整行 + 中文品名写「未命中：原因」（型号/尺寸/材质/关键词搜不到四选一或组合）；待定 → 中文品名标「待定：原因」；分类：紧固件命中=CONSUMABLE、未命中=dead inventory、命中但非紧固件=留空。**图片路径（用户 2026-09-15 19:30 确认：后续路径都需修改）**：本轮 NO 1-10 行图片链接统一修正为单反斜杠绝对路径 `=HYPERLINK("D:\sara\库存管理\图片\W1-SHELF4\<n>.jpg","photo link")`（替换现有相对路径 `实拍图/W1-SHELF4/<n>.jpg`）；SHELF4 其余行及后续货柜的相对路径按同规则在后续批次一并修正；改前备份。
- anti_lazy_checklist:
  - [x] 不会按行号/族批量假设品类（每行逐张识图）
  - [x] 有图必识图（超大图先缩放，NO 1-10 图全部逐张看）
  - [x] 发票检索覆盖词组+单词+缩写，不偷懒只搜一个词（先补字典再全量查）
  - [x] 每 batch_size 件停一次交 batch_ready（NO 1-5 交一次，NO 6-10 再交一次）
  - [x] 未批准前不写主表（当前只做只读探查，未动 Excel）

### Cursor 计划审批（Cursor 填）

- plan_verdict: **approved**
- plan_checked_at: 2026-09-15 19:32
- plan_notes: |
    范围/批次/识图/字典先行/全量检索/写回规则齐全，anti_lazy 已勾。
    Cursor 复核：`图片\W1-SHELF4\1.jpg`–`10.jpg` 均存在，尺寸均 ≤2000px，本批可不缩放。
    字典现状：有 VELOX/TAPPO/RACCORDO；**缺** CAMLOCK、GEKA、GUILLEMIN、独立 TUBO/FLEX（仅有 REGGITUBO）——按计划先补再匹配。
    NO6 物料号 0900567 非 PEN：命中判定必须优先死库存规则，识图仅作复核。
    图片链接本轮保持原样、不擅自全表改相对路径——同意。
    **批准开工：先做批1 NO 1–5**，做完立刻 `batch_ready`，勿连做 6–10。
- plan_issues:
  1. （无阻断项）交检时须附：每行识图一句话结论、检索关键词列表、字典新增词条名。

---

## ② 当前小批进度（豆包每批填写）

- batch_index: 1
- batch_rows: W1-SHELF4 NO 1-5（数据行 R4-R8）
- batch_done_actions:
  1. 补字典 6 条（其他字典（待定）SHEET 末尾）：TUBO / TUBO FLESSIBILE / DIRITTO / GEKA / GUILLEMIN / OTTONE（各含原文/扩展名/中文/同义词/图片，图片=W1-SHELF4 实拍图）
  2. 逐张识图 NO 1-5（图片均 ≤2000px 直接 Read）：
     - NO1: 黑色圆柱形工业件，顶部吊耳+两侧金属连接臂，模压字符无法辨认 → 待定（需用户确认型号/用途）
     - NO2: 不锈钢金属编织软管，两端直口焊接接头 → 死库存
     - NO3: 不锈钢金属编织软管（记录 316 90° 终端）→ 死库存
     - NO4: 黄铜 Camlock D 型快速接头（铸字 D-72，记录 D75）→ 死库存
     - NO5: 黄铜卡扣式快速接头（带手柄+黑色密封胶圈），无尺寸 → 待定
  3. 发票全量检索（invoice_full_dump.txt 793 行 + 物料发票表20260911.xlsx 109 SHEET + 近3年表）：
     - 关键词：CAM-LOCK/CAMLOCK/VELOX/RACCORDO/TAPPO/TUBO/TUBO INOX/FLESSIBILE/FLEX/TRECCIATO/METALLICO/TERMINALE/OTTONE/GUILLEMIN/GEKA，词组+单词+缩写全量
     - 结果：不锈钢编织软管（TUBO INOX FLESSIBILE/TRECCIATO）发票全库 0 匹配；黄铜快速接头仅 VELOX 系列（I.S.I. RACCORDO OTTONE VELOX 1½" GAS F / TAPPO OTTONE VELOX；MALDOTTI PORTAGOMMA RAPIDO VELOX 7/8" OTTONE、VELOX FEMMINA 1" 1½" 等，最大 1½"）
  4. 判定写回（备份 `库存未匹配_备份_20260915_193831.xlsx`）：
     - NO1: 待定（中文品名标注需用户确认）
     - NO2: dead inventory 淡蓝，原因「发票无不锈钢编织软管品类」
     - NO3: dead inventory 淡蓝，原因同上（316 90° 终端）
     - NO4: dead inventory 淡蓝，原因「黄铜快速接头仅 VELOX 1/4"-1½"，无 3"(D75)，同族仅尺寸不同」
     - NO5: 待定（缺尺寸）
     - 图片链接：NO 1-5 已按用户 2026-09-15 19:30 指令统一为单反斜杠绝对路径（Cursor 审批时写"本轮保持原样"，但用户指令优先，已改，特此说明）
- backup: D:\sara\库存管理\库存未匹配\库存未匹配_备份_20260915_193831.xlsx
- dict_changed: yes（+6 条）
- evidence: 识图结论见上；NO4 OCR 铸字「D-72」（记录 D75，误差可接受）；发票无命中行，未动近3年已命中 SHEET

- fix_round (回应 Cursor issues 2026-09-15 19:45):
  - issue1（NO1 识图定性不足）：已重识图（裁剪放大 4 区域实读），确认 = 黑色 PP-V Camlock 防尘盖（Tipo DC）：圆柱盖体+顶部提手+两侧卡扣耳；侧面模压字 OCR 读「volvo」/Cursor 读「60-100」（磨损），按「60-100」≈适配 2"-3" 处理。补字典词条「DC」（Dust Cap/Tappo Femmina Cam-Lock）。全量检索 CAM-LOCK TIPO DC：PP-V NERO DC 1½"/2"/3"（PEN2304/2305/2306）、316 DC 2"/3"（PEN3337/3338）、316 3/4"（DED0002 死库存）、316 3"（PEN2296）。结论：**待定**——需用户补测精确尺寸/内径后命中 1½"/2"/3" 对应规格。Product Name 已写 RACCORDO CAM-LOCK TIPO DC PP-V NERO，中文品名已注明候选规格与待测原因。
  - issue2（NO4 文案不实）：中文品名已改为「已检索 CAMLOCK/CAM-LOCK/TIPO D（发票 DC/A/D/F 型多为 INOX 316 与 PP-V 塑料，未见黄铜 OTTONE 材质 D75/3" 规格），同族仅尺寸/材质不同」。判定保持死库存+淡蓝。
  - issue3（批2 未动）：NO 6-10 未触碰，等本批复检通过再做。
  - 备份：库存未匹配_备份_20260915_194551.xlsx


- batch_index: 2
- batch_rows: W1-SHELF4 NO 6-10（数据行 R9-R13）
- batch_done_actions:
  1. 备份：`库存未匹配_备份_20260915_200303.xlsx`
  2. 逐张识图 NO 6-10（图片 ≤2000px 直接 Read）：
     - NO6: 黄铜卡扣式快速接头（两卡扣耳+黑橡胶密封圈+宝塔口），刻字 **7/8"**；**图文矛盾**：记录品名 GUILLEMIN INOX ø080x3" 为不锈钢 3" 内丝，实物黄铜 7/8" → 已提醒用户校对
     - NO7: 银灰金属快速接头（六角头+螺纹孔+卡扣耳座，△铸标）→ VELOX F 1" 母头
     - NO8: 银灰金属堵盖（卡扣耳+环形密封+挂孔/链孔）→ 疑似 Camlock DC 带链孔堵盖 vs 记录 Velox tappo，材质待确认
     - NO9: 黄铜圆盘件（中心凸台+两侧耳），铸字 **PT 130 / 700 / 1½**（反向），供应商 RIDART
     - NO10: 金属圆盘件（中心凸台+上下耳），无 OCR 文字，与 NO9 同族倾向
  3. 发票全量检索（物料发票表 + 近3年表 + invoice dump）：
     - RIDART：**109 SHEET 全库无 RIDART**（两表 0 命中）→ NO9 供应商无发票记录
     - 1½" 200 bar 黄铜/高压件：全库 0 匹配
     - MALDOTTI VELOX：PEN1538 RACCORDO RAPIDO VELOX FEMMINA 1"（RF1）→ **NO7 命中**；TAPPO RAPIDO VELOX PEN1623（黄铜带链，配图 137.jpg）→ NO8 候选
     - I.S.I. GEKA/GUILLEMIN：仅 PEN2234 GUILLEMIN INOX FEMMINA ø080x3"（0900567）→ NO6 发票条目存在但**与实物黄铜 7/8" 不符**
  4. 判定写回（备份后）：
     - NO6: **dead inventory 淡蓝**——条码 0900567 非 PEN（规则优先）+ 实物黄铜 7/8" 快速接头发票无匹配；中文品名注明图文不符需用户校对
     - NO7: **命中 PEN1538**（MALDOTTI，RF1，单价 3.187333，税 22）——写发票信息+近3年已命中 SHEET 新增序号177（含供应商）；非紧固件分类留空
     - NO8: **待定**——TAPPO RAPIDO VELOX（PEN1623）材质（记录 Inox vs 发票黄铜）与结构待用户确认
     - NO9: **dead inventory 淡蓝**——RIDART 发票全库无记录 + 1½" 200 bar 黄铜全库 0 匹配
     - NO10: **待定**——缺尺寸/型号，需补测后检索
  5. 图片链接 NO 6-10 已改单反斜杠绝对路径（用户 19:30 规则）
  6. 补字典 4 条（其他字典 SHEET）：FEMMINA / MASCHIO / RAPIDO / RACCORDO（含图片 HYPERLINK）
- backup: D:\sara\库存管理\库存未匹配\库存未匹配_备份_20260915_200303.xlsx
- dict_changed: yes（+4 条）
- evidence: 识图结论见上；NO6 OCR 7/8"；NO9 铸字 PT 130/700/1½；近3年已命中 SHEET 已新增 PEN1538 行（序号177，含供应商 MALDOTTI）
- issues_for_user:
  1. **NO6 图文/条码矛盾**：主表品名 GUILLEMIN INOX ø080x3"（发票 PEN2234，条码 0900567 真实存在）但实物为黄铜 7/8" 卡扣式快速接头——可能货位/条码错配，请用户校对
  2. NO8 材质待确认（记录 Inox vs 发票 TAPPO RAPIDO VELOX 黄铜带链）
  3. NO9 RIDART 供应商发票近3年无任何记录（全库 0 SHEET）——请确认供应商名是否写错

### Cursor 小批检查结果（批1 · 归档）

- verdict: **pass**
- checked_at: 2026-09-15 19:50
- checked_rows: NO 1、2、4（复检主表写回）
- summary: |
    复检通过。`image_reject_count: 1`（批1识图驳回配额已用完，不得再因识图 fail 批1）。
    NO1：已改为 CAM-LOCK TIPO DC PP-V NERO + 待定补测尺寸，检索候选 PEN2304/2305/2306 表述合理。
    NO4：死库存原因已改为「已检索 CAMLOCK…未见黄铜 OTTONE D75」，文案合规。
    NO2/3 淡蓝死库存、NO5 待定保持可接受。
    **当时批准进入批2：NO 6–10**。
- issues:
  1. （无阻断）NO1/5 待定项留给用户补尺寸；记入提醒即可。
- next_action: **batch_continue**（历史）


- fix_round (回应 Cursor issues 2026-09-15 21:33，备份 `库存未匹配_备份_20260915_213723.xlsx`):
  - issue1（NO6 文案不实/不得用发票品名覆盖实物）：已改。Product Name 改回原表 `Raccordo GEKA`（识图=黄铜 7/8" 卡扣+宝塔口）；中文品名三句齐：①条码 0900567 非 PEN → 规则死库存（即使发票有近似规格也不改判命中）；②该条码指向 I.S.I. PEN2234 GUILLEMIN INOX FEMMINA ø080x3"（不锈钢 3" 内丝），与实物黄铜 7/8" 不符，请用户校对货位/条码；③实物近似批1已检出的 MALDOTTI PORTAGOMMA RAPIDO VELOX 7/8" OTTONE（DED0008），因条码规则不改判命中。已删除「实物黄铜 7/8" 快速接头发票无匹配」矛盾句。保持 dead inventory+淡蓝。
  - issue2（NO9 补全量检索证据）：已补全量检索（invoice_full_dump + 物料发票表 109 SHEET + 近3年表），关键词 RIDART / PT130 / PT 130 / PT-130 / 700 / 200 bar / 200BAR / 200 BAR / 1½" / 1 1/2" / 1.5"：**RIDART 0 命中（全库无此供应商 SHEET）；PT130/PT-130 0；700 仅 STRENX 700 MC 钢板（与黄铜接头无关）；200 bar 仅 ARCO GAS 气体气瓶；1½" 相关为 PVC 变径/304 球阀/管箍/双头对丝，无黄铜高压件；1.5" 0**。结论 0 匹配 → 保持 dead inventory+淡蓝，中文品名改为「未命中：已检索 RIDART/PT130/700/1½"/200 bar，发票无；记录 200 bar 与铸字 700 不一致，请用户校对」。
  - issue3（非阻断）：NO8/10 待定与 NO6 条码错配继续 issues_for_user；本批仅动 NO6/NO9，批1已通过行（NO1-5）未触碰。
  - evidence: NO9 检索关键词与 0/命中明细见上。

### Cursor 小批检查结果（批2 初检 · 归档）

- verdict: **fail**
- checked_at: 2026-09-15 21:33
- checked_rows: NO 6、7、8、9、10（云端无 Excel/实拍图，以 bridge 证据抽查）
- image_reject_count: 0（本批未因识图驳回；下列为写回文案 / 检索硬伤）
- summary: |
    备份路径、逐行识图痕迹、字典 +4、NO7 命中 PEN1538、NO8/10 待定方向可接受。
    **NO6 死库存结论正确**（条码 0900567 非 PEN，规则优先），但写回/原因文案不实，且疑把条码对应的 GUILLEMIN INOX 3" 写到实物黄铜 7/8" 行上。
    **NO9** 铸字 PT 130 / 700 未见检索词列表，属全量检索证据不足。
    NO7 命中 MALDOTTI VELOX FEMMINA 1"（PEN1538）+ 近3年已命中序号177、非紧固件分类留空：通过。
    NO8 待定（记录 Inox vs 发票黄铜 TAPPO RAPIDO VELOX PEN1623）：通过，留给用户。
    NO10 待定缺尺寸：通过；与 NO9 同族倾向可在 NO9 补检索后于 reasons 中交叉引用，不必强行死库存。
- issues:
  1. **NO6（必须改）**：保留 **dead inventory + 淡蓝**，但禁止用条码发票品名覆盖实物。
     - Product Name / 中文品名以识图为准：黄铜 7/8" 卡扣+宝塔口（原表 `Raccordo GEKA` 7/8" 更接近），**不要**写成 GUILLEMIN INOX ø080x3"。
     - 原因须同时写清三句：①条码 0900567 非 PEN → 规则死库存（即使发票有近似规格也不改判命中）；②该条码指向 I.S.I. PEN2234 GUILLEMIN INOX FEMMINA ø080x3"，与实物不符，请用户校对货位/条码；③实物近似批1已检出的 MALDOTTI `PORTAGOMMA RAPIDO VELOX 7/8" OTTONE`，因条码规则不改判命中——删除「实物黄铜 7/8" 快速接头发票无匹配」这种与批1检索结果矛盾的句子。
  2. **NO9（必须改）**：补全量检索铸字与高压词，并把**关键词列表 + 0/命中**写进 evidence。至少覆盖：`RIDART`、`PT130`/`PT 130`/`PT-130`、`700`、`200 bar`/`200BAR`/`200 BAR`、`1½"`/`1 1/2"`/`1.5"`。若仍 0 命中：保持死库存+淡蓝，中文品名改为「未命中：已检索 RIDART/PT130/700/1½"/200 bar，发票无；记录 200 bar 与铸字 700 不一致，请用户校对」。
  3. （非阻断）NO8/10 待定、NO6 条码错配继续 `issues_for_user`；修正后只交本批，勿动批1已通过行。
- next_action: **needs_doubao_fix**（历史）

### Cursor 小批检查结果（批2 复检 · Cursor 填）

- verdict: **pass**
- checked_at: 2026-09-15 21:45
- checked_rows: NO 6、7、8、9、10（云端无 Excel/实拍图，以 bridge 证据复检 round 2）
- image_reject_count: 0（本批未因识图驳回）
- summary: |
    复检通过。备份 `库存未匹配_备份_20260915_213723.xlsx`；仅动 NO6/NO9，批1未触碰。
    **NO6**：PN 改回 `Raccordo GEKA`；中文品名三句（非 PEN 死库存 / 条码指向 PEN2234 GUILLEMIN INOX 3" 与实物黄铜 7/8" 不符请校对 / 近似 MALDOTTI VELOX 7/8" OTTONE 因条码规则不改判）齐备；已删矛盾句；dead inventory+淡蓝。
    **NO9**：已列 RIDART/PT130/PT 130/PT-130/700/200 bar/200BAR/200 BAR/1½"/1 1/2"/1.5" 及 0/无关命中；死库存原因含铸字 700 vs 记录 200 bar 不一致。
    NO7 PEN1538、NO8/10 待定维持通过。
    计划范围 W1-SHELF4 NO 1–10 两批均过 → **done**。
- issues:
  1. （无阻断）留给用户：NO1/5 待测尺寸；NO6 货位/条码错配；NO8 材质（Inox vs 黄铜）；NO9 RIDART/铸字；NO10 缺尺寸。
- next_action: **done**

---

## 操作规则（两边必守）

1. **先计划后动手**：新任务必须 `plan_submitted` → Cursor `plan_approved` 后才能写 Excel。
2. **小批交检**：默认每 **5** 件（`batch_size`）停一次 → `batch_ready`；未抽查通过不得做下一批。
3. 一次只有一个 `owner`；改状态同时改 `updated_at`。
4. Cursor 抽查重点：是否真识图、是否全量检索痕迹、字典是否先补、确认列/淡蓝/原因是否合规。
5. 发现「未识图就定性 / 只搜一词 / 一批做完才汇报」→ 直接 `fail` + `needs_doubao_fix` 或 `plan_rejected`。
6. 需要用户决定 → `blocked`。
7. **识图驳回上限（用户 2026-09-15 19:46 锁定）**：豆包已对某批做过错识图交检后，Cursor **因图片/识图原因驳回最多 1 次**（同一 `batch_index` / 同一批行）。豆包按 issues 修正并再次 `batch_ready` 后：
   - Cursor **不得再以识图/看图分歧**为由二次 `fail`；
   - 若仍有图文疑义 → 记入 `plan_notes`/`summary` 提醒用户，或 `blocked` 请用户拍板，**允许本批因识图争议放行进入下一步**（检索造假、未备份、未写淡蓝/原因、违反条码规则等非识图硬伤仍可驳回，不受本条 1 次上限约束）。
    - 交检记录建议写：`image_reject_count: 0|1`（本批因图已驳回次数）。
