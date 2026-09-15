# 豆包 ↔ Cursor 协同协议（先计划 · 小批抽查）

更新：2026-09-15

目标：防止偷懒（未全量查发票、未识图、按族瞎判）。流程固定为：

```
豆包交计划 → Cursor 批计划 → 豆包做一小批(默认5件)
  → Cursor 抽查 → 通过则下一批 / 不通过则返工
  → 全部批次通过 → done
```

互通文件：`D:\sara\库存管理\handoff\bridge.md`  
流程细则仍服从：`workflow.md`、`inventory_handoff_context.md`

---

## 给用户：怎么跟豆包说（可复制）

请按 `handoff/bridge_protocol.md` 与 Cursor 协同，只用 `handoff/bridge.md`。

硬规则：
1. **每次动手前**先在 bridge 写清「执行计划」（怎么识图、怎么全量查发票、本批范围），`status=plan_submitted`，等 Cursor 批准。**未批准禁止改主表**。
2. 批准后一次只做 **5 个产品**（或 bridge 里的 batch_size），做完立刻 `status=batch_ready` 交检，**禁止攒一大批发**。
3. 看到 `needs_doubao_fix` / `plan_rejected` 按 issues 改；看到 `batch_continue` 再做下一批。
4. 抽查会查：有没有逐张识图、检索是否词组+单词+缩写全覆盖、字典是否先补。

---

## 豆包侧流程

### A. 新任务 → 交计划（不能偷懒写空话）

在 `bridge.md`「① 执行计划」填全：

- 精确行号范围
- **识图方法**（超大图缩放路径；承诺逐行看图）
- **字典**：先补哪些词；匹配用词组+单词+缩写
- **发票全量检索**：查哪些源、关键词怎么组合；写明「不是只搜一个词」
- **写回规则**：命中/未命中/待定怎么落列
- anti_lazy_checklist 全部勾 yes

然后：

```
status = plan_submitted
owner = cursor
```

**停。等 Cursor。**

### B. 计划通过 → 做一小批

收到 `plan_approved` / `batch_continue`：

1. 只处理下一批 ≤ `batch_size` 行
2. 每行：识图（如需）→ 字典 → 全量检索 → 再写表
3. 备份；changelog 可按批或按任务追加
4. 填写「② 当前小批进度」
5. `status=batch_ready`，`owner=cursor`，`batch_index+=1`

### C. 返工

`needs_doubao_fix`：只改 issues 点名项 → 再 `batch_ready`。  
`plan_rejected`：改计划 → 再 `plan_submitted`（仍不能写表）。

---

## Cursor 侧怎么检查

### 1) 审计划（`plan_submitted`）

不通过则 `plan_rejected`，常见理由：

- 未写清「逐行识图 / 超大图预处理」
- 检索计划只有单关键词，无词组+单词+缩写
- 范围过大却声称一次做完、不设 batch
- checklist 未勾或空泛

通过：`plan_approved`，`owner=doubao`。

### 2) 小批抽查（`batch_ready`）——隔几个产品查一次

对**本批全部或抽样 ≥3 行**（批≤5 则尽量全查）：

| 查什么 | 如何判偷懒 |
|--------|------------|
| 识图 | 有图行是否有识图结论/尺寸依据；图文明显不符却直接定性 |
| 字典 | 新意语 PN 是否能在字典找到或本批已声明新增 |
| 全量检索 | evidence/changelog 是否体现多关键词；原因是否像「随便搜一下」 |
| 写回 | 命中/未命中/待定与淡蓝、条码规则是否一致 |
| 图片链接 | 单反斜杠 HYPERLINK |

- pass 且还有剩余行 → `batch_continue`
- pass 且范围做完 → `done`
- fail → `needs_doubao_fix` + 具体 NO 与改法

### 3) 默认不直接大改主表

问题写回 bridge，让豆包改。

---

## Cursor 循环建议

监视 `bridge.md`：

- `plan_submitted` → 审计划  
- `batch_ready` → 小批检查  
- 其它状态 → 等待  

用户说「开始监视 bridge」后再开循环。

---

## 限制

- 两边都要读 bridge；只开 Cursor 不够  
- 这是防偷懒流程，不是替代 `workflow.md` 专业规则  
