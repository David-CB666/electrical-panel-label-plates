---
name: electrical-panel-label-plates
description: 机电工程「电箱标签牌」全自动制作技能。当用户要从 CAD 单线图（SLD / DWG / DXF）读出配电箱回路表并生成黑底白字 Word 标签牌（贴在每个断路器 MCB/RCD 下方的用途+回路编号条），或要由 Word 标签牌转成带尺寸标注的 CAD 向量加工图（DXF / SVG）给广告商/加工场 1:1 落料时使用。触发词：电箱标签、配电箱标签牌、回路标签、面板标签、MCB 标签、RCD 标签、单线图回路表、电气标签牌、标签牌 CAD 加工图、label plates、panel schedule to docx / to dxf。覆盖两种 SLD 网格格式（A 纵向量表 / B 横向网格），含 RCD 极数人工确认门槛（17:25 斜线数原则）、fixed-layout 栏宽锁定、verify gate。只出 Word 不出 PNG；可加 CAD 向量加工图。
---

# 电箱标签牌制作技能（electrical-panel-label-plates）

## 0. 这个技能做什么

机电工程中「电箱标签牌」= 贴在配电箱（电箱）内每个断路器下方的小标识条，黑底白字，写「用途 + 回路编号」。广告商据此印制后贴到现场每个 MCB/RCD 上。

本技能把流程标准化、脚本化：

```
CAD 单线图 (DWG) → DXF → 解析回路表 (JSON) → 人工确认 RCD 极数 → 生成 Word 标签牌 (docx)
                                                                      ↓（可选 --cad）
                                                           CAD 向量加工图 (DXF / SVG，带尺寸标注)
```

**知识库 canonical 路径（脚本全在这里，本技能只引用、不复制代码）：**
`<KB_DIR>/electrical-panel-labels/`
- 脚本：`scripts/` 子目录（30 个，见 §7 索引）
- 文档：规格文档 00~08 + README
- 回路表 JSON：示例项目电箱（多个版本）

**新任务直接照 §6 工作流做，不要重新发明。**

---

## 1. 触发条件

- 用户给出 DWG / DXF 单线图，要生成「电箱标签牌 / 配电箱标签 / 回路标签 / MCB 标识条」。
- 用户已有回路表 JSON，要重新生成或调整 Word 标签牌。
- 用户要由 Word 标签牌转 CAD 加工图（带尺寸标注的 DXF / SVG，给广告商/加工场落料）。
- 用户问「这个 RCD 是几 P」「栏宽对不对」「标签牌格式」「标签牌 CAD 图」等。

**不适用**：若用户只要「电箱大样图 / arrangement drawing」→ 改用 `cad-sld-to-arrangement` 技能。若用户要「材料报批」→ 改用 `macau-material-approval`。

---

## 2. 两条用户铁律（最高优先级，违反即返工）

1. **只出 Word（.docx），不出 PNG 图。** 除非用户明确要求图片（如给广告商的规格标注图特例）。
2. **用户手改过的 docx 只准 python-docx 读取理解，禁止覆写。** 重新生成时输出必须去 `<TEMP_DIR>/` 或加 `_v2`/`_修正版` 后缀，绝不直接覆盖用户已确认的文件。

---

## 3. ⚠️ 铁律：RCD 极数 = 单线图符号「斜线数」（17:25 原则，通用）

> 这是本技能最容易犯、也最致命的错误。

**RCD（漏电断路器）是几 P（极），一律以单线图 RCD 符号里的「斜线数」判定，绝对不可以按 rating（额定电流/漏电电流）硬推算或固定。**

- 例如 `25A / 300mA` 可以是 **2P** 也可以是 **4P** —— 逐图看符号斜线数决定，不可当成固定值。
- `30mA` 多数 2P、`300mA` 多数 4P 只是**粗略 heuristic，仅供快速估计，最终以符号斜线数 + 用户确认为准**。
- **馈线 MCCB（P1 / P2 / 备用）是例外**：不用符号斜线规则，改用 DXF 三相位置法判定（三个等距 32A 文字在三相位置 = 3P）。

**强制流程保障（human-in-the-loop）**：解析器输出 `pole_status = "NEEDS_HUMAN_REVIEW"`，编排器在生成前会拦截，要求先跑 `--review-poles` 人工确认。详见 §5。

---

## 4. 输出规格（标签牌）

| 项目 | 值 |
|---|---|
| 底色 / 字色 | 黑底 `000000` / 白字（自动）/ 白边框 0.5pt（sz=4）|
| 字型 | SimHei（黑体）|
| 行高 | 上行用途 1758 twips（3.1cm）/ 下行编号 510 twips（0.9cm），exact |
| 栏宽（必须 `tblLayout=fixed`）| MCB 1P = 1.8cm；RCD = 1.8×P（2P=3.6cm、4P=7.2cm）；馈线 3P = 5.4cm |
| 纸张 | 回路数 > 22 用 A3 横向（42×29.7cm），否则 A4 横向；边距 1.27cm |
| 结构 | 每个 RCD 配一组（含其下 6~12 个 MCB 回路栏），备（RE）全显示；无 RCD 的尾组单独成表 |

**栏宽锁死靠 `tblLayout=fixed`**：否则 Word 会把每张表自动拉到填满页宽，令 3P/1P/4P 的宽度差在不同表之间看不出分别。生成器已清除 `table.autofit=False` 自动注入的重复 `tblLayout`，并对每格清旧 `tcW` 再加。

---

## 5. 两阶段工作流（编排器入口）

**唯一入口**：`make_panel_labels.py`（在 KB `scripts/` 下）。它自动：解析输入 → 判断 SLD 格式 → （可选）印极数确认表 → 生成 docx → 强制跑 verify gate → （可选）生成 CAD 向量加工图。

### 阶段一：解析 + 人工确认极数（不直接生成）
```bash
# 从 DWG 解析并只印 RCD 极数待确认表（17:25 原则），不生成
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --review-poles

# 用户按单线图符号斜线数核对后，确认写回 JSON（pole_status=CONFIRMED）
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --review-poles --review-confirmed
```
`--review-poles` 会列出每个面板的 RCD 及其「建议极数」，并提醒「请按图符号斜线数确认」。加 `--review-confirmed` 才把 `CONFIRMED` 写回 JSON（格式 B 同时按 mA 写 `rcd_poles`）。

### 阶段二：生成（极数须已确认）
```bash
# JSON 已确认 → 直接生成 Word
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 回路表.json --out 标签.docx

# 自动判格式 + A3 纸
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --paper A3

# 生成 docx 同时产 CAD 加工图（DXF + SVG，带尺寸标注）
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 回路表.json --out 标签.docx --cad --cad-svg
```
**门禁**：若 JSON 仍有面板 `pole_status == "NEEDS_HUMAN_REVIEW"` 且未加 `--force`，编排器直接 `sys.exit` 拒绝生成（强制走人工确认）。生成后**强制**跑 `verify_label_widths.py --gate`，栏宽不在 `{1.8, 3.6, 5.4, 7.2}cm` 即报错退出。

> `--force` 跳过极数门禁（不推荐，会绕过 17:25 保护）。`--skip-verify` 仅 debug 用。
> `--cad` 在 docx 生成后调用 `gen_label_cad.py`；`--cad-out` 指定输出路径（`.dxf`/`.svg`），默认 = docx 同名 `.dxf`；`--cad-svg` 同时多写一份 `.svg`。

---

## 6. 标准工作流（新任务照做）

```
1. 收到 DWG 单线图
   ↓
2. DWG→DXF（accoreconsole，须用 ASCII 临时路径，中文路径会静默失败；stdout 是 UTF-16LE 勿在 Python subprocess 读）
   ↓
3. explore_dxf.py 探索结构 → 判断 SLD 格式
   - 有 "N CIRCUITO" → 格式 B（横向网格）→ parse_el01.py
   - 有「编号/用途/功率(kVA)」中文行 → 格式 A（纵向量表）→ parse_sld_panels.py
   ↓
4. 解析 → 回路表 JSON（含 RCD 分组 + pole_status=NEEDS_HUMAN_REVIEW）
   ⚠️ 分组铁律见 §8；极数见 §3（17:25）
   ↓
5. 交叉核对：找同目录「落实版 PDF」，fitz 提取文字核对（用 --pdf 可自动比）
   ↓
6. 阶段一：--review-poles 印极数表 → 用户按斜线数确认 → --review-confirmed 写回
   ↓
7. 阶段二：--input 回路表.json 生成 Word 标签牌（只出 Word！）
   ↓
7.5 （如需 CAD 加工图）编排器加 --cad [--cad-svg]，或单独跑 gen_label_cad.py --docx 标签.docx --out 标签.dxf [--svg]
   ↓
8. 交付：docx（必出）+ dxf/svg（加工用，按需）放用户任务资料夹；同时沉淀回路表 JSON + 脚本到知识库
```

---

## 7. 脚本索引（KB `scripts/` 下，编排器已串好路径）

| 脚本 | 用途 |
|---|---|
| `make_panel_labels.py` | **编排入口**（解析→格式判断→极数确认→生成→verify gate→可选 CAD）|
| `parse_sld_panels.py` | 格式 A（纵向量表）解析器 → JSON（`--dxf`/`--out`）|
| `parse_el01.py` | 格式 B（横向网格）解析器 → JSON（`--dxf`/`--out`）；v4 含 `group_by_rccb_span()` 几何跨度量度分组 |
| `gen_panel_labels_圖書館_v2.py` | 格式 A 生成器（`--json`/`--out`/`--paper`）|
| `gen_label_docx_el01.py` | 格式 B 生成器（`--json`/`--out`/`--paper`）|
| `verify_label_widths.py` | 栏宽校验 gate（`--docx`/`--gate`，退出码 1=不合规）|
| `gen_label_cad.py` | **Word → CAD 向量加工图**（DXF/SVG 带尺寸标注；读 docx 栏宽/行高 1:1 还原）|
| `verify_label_cad.py` | ⭐ **CAD 加工图验证器**：查每格 ≤2 行 + DXF/SVG 全部文字入框（正常行距 + 行距 2x 最壞情況）；`--glob` 避中文路径坑 |
| `regen_label_cad.py` | ✅ **重生成驱动**：glob 解析 docx 路徑，一句重生成 DXF+SVG（RCD 欄**自動合併**，唔使 flag；`--no-svg`）|
| `render_check_label_cad.py` | ✅ **無頭自渲染自檢**：ezdxf→PNG 自渲染 + 幾何自檢（入框/居中/重疊/相撞/RCD 行數），唔使等用戶截圖；`--glob`/`--no-render`/`--png` |
| `dx2dwg_scr.py` | ✅ **DXF→DWG 交接地**：由 DXF 生成 AutoCAD Script (.scr)，可 `--console` 自動搵 accoreconsole 無頭轉檔出 .dwg |
| `tests/test_label_cad.py` | ✅ **pytest regression gate**：自製 fixture docx 生成 DXF+SVG 後跑全部幾何自檢 + 驗證 RCD auto-merge 真·合併；`python -m pytest tests/ -q` |
| `gen_big_labels_word.py` | 电箱名牌大标签 Word（80×50mm，48pt 两行，用户确认版）|
| `explore_dxf.py` | DXF 结构探索（诊断）|
| `explore_feeder_p1p2.py` / `render_feeder.py` | 馈线 MCCB 三相位置法极数判定 |
| `analyze_rccb_span*.py` / `explore_rccb_geo.py` | RCD 几何跨度量度调试 |
| 其余 | 早期探索/测量脚本，新任务一般不需直接调用 |

**执行环境**：须用装有 `ezdxf` + `python-docx` + `PyMuPDF` 的 Python。本机用：
`<CAD_PYTHON>`
编排器默认取 `PYTHON_EXE` 环境变量，否则用运行本文件的解释器。

---

## 8. ⚠️ 分组铁律（格式 B 最重要，取代固定 6 个规则）

每个 RCD 的回路数 = **RCCB 在图中横跨的竖线范围内上方连接的回路 MCB 数**（一般 6~12、大电箱 6~9，**无固定数，不准硬编 6**）。

- **Primary = 几何跨度量度**（`group_by_rccb_span()`）：① 收集 RCCB 行竖线；② 按 x 间距分组（gap > max(1.5, 栏距×1.5)）；③ 每组竖线数 = 回路数；④ 竖线总数 = 回路数才算对齐成功，否则 fallback。
- **Fallback = RCCB 文字锚点对齐**（实测 x 对齐群组内第 4 个回路栏，Δx≈-0.07 → `circuits[anchor_idx-3 .. anchor_idx+2]`）。
- **不要用「每个回路配右边最近 RCCB」**（会令第一组只得 4~5 个）。
- **单一主 RCD 例外**：电箱只有 1 个 RCCB → 覆盖全箱所有回路。备（RE）都要入组显示。
- **连续无 RCD 组合并为单一尾组。**
- 面板间文字污染坑：RCCB 文字(mA)在不同面板有相近 y，须用各面板精确 `PANEL_Y` 分离。

---

## 9. 踩过的坑（绝对不要再踩）

### Word/排版类
1. **twips 换算**：1cm = 567 twips。80mm = 8cm = 4536（不是 `int(80*567)`=45360，那是 80cm！）。50mm = 2835。
2. **OOXML 唯一元素要先清再加**：`w:tcW`、`w:tblGrid` 等 schema 规定只一个，不清旧就 add 会出现两个值，Word 用错那个。
3. **run rPr 用高层 API**：`run.bold` / `run.font.size` 会自动创建 rPr；别用 `OxmlElement('w:rPr')` 从零搭（schema 顺序问题会令 run 完全空白）。
4. **`cell.text=''` 怪行为**：会保留第一个空 paragraph，后续 `add_run` 可能失效。安全做法：彻底删所有 paragraphs 再 rebuild。
5. **浮動表格 + tblLayout=fixed + tblGrid 明确 gridCol widths** 才锁得住栏宽（48pt 中文字才不爆框）。

### 流程/规矩类
6. 只出 Word 不出 PNG（§2）。
7. 用户手改过的 docx 禁止覆写（§2）。
8. 电箱名牌大标签格式（用户确认）：80×50mm、两行都 48pt 粗体、名称统一「配电箱」（不带位置后缀）、总开关 64pt 粗体单行、无规格说明栏。

### SLD 解析类
9. SLD 两种排版格式（§6 步骤 3 判断）：A 纵向量表 / B 横向网格。
10. 格式 A：DWG 文字用 `align_point`（群码 11/21/31）非 insert，差 4 twips 会抢配对 → greedy unique 配对（text 不重用 + 有 kVA 跳过「备用」）。
11. 格式 B：`x_nc` 必须用「N CIRCUITO」文字本身的 x（别用 y 带 min x——会拿到左边「A-1-01」等文字）；RCCB 标记要独立取（别用每个回路最近邻，会碎成 20+ 组）。
12. 分组铁律见 §8。**RCD 极数见 §3（17:25 斜线数原则），绝不可按 rating 硬编。**

### CAD 转档类
13. DWG→DXF：accoreconsole `_DXFOUT` 到 **ASCII 临时路径**（中文路径会静默失败）；stdout 是 UTF-16LE，别在 Python subprocess 读。
14. ezdxf 只读 DXF 不读 DWG（先转 DXF）；COM 并发先 `taskkill /F /IM acad.exe` + sleep。
15. **DXF 尺寸标注**：用 `msp.add_linear_dim(...).render()`；绘制单位设 `INSUNITS=5`（cm）令标注量度值直接显示 cm。dimstyle 设 `dimtxt`/`dimgap`/`dimasz`/`dimlunit=2`/`dimdec=2`。
16. **⚠️ DXF 文字出框（最高優先）**：**絕對唔可以用「整欄一個 MTEXT + 空白行推落 + 擺板垂直中心 + MIDDLE_CENTER」**。呢招 AutoCAD 實際渲染時 `line_spacing_factor` / `MIDDLE_CENTER` 解讀差異會令成塊字高出框外。**正確做法 = 逐 row 一個 MTEXT、垂直置中喺該 row band 內**。每 row ≤2 行 → 字塊 ≤0.96cm ≪ row_h，絕對入框。SVG 天生逐 row 定位無事。驗證用 `verify_label_cad.py`（座標解析，唔靠目測）。
17. **⚠️ 路徑字符坑**：用戶資料夾路徑如果有特殊字/罕用字，**唔好喺 bash 手打寫死路徑**（shell Unicode 編碼會靜默失敗，cp/test 都無反應）；要用 `glob.glob()` 喺 Python 內部解析（見 `regen_label_cad.py` / `verify_label_cad.py` 嘅 `--glob`）。
18. **⚠️ verify 入框檢查會「假 pass」**：整張圖框（最外圍 sheet border）都係 color=1 LWPOLYLINE、面積最大，會包住所有文字。`_plates_from_dxf` 必須剔走面積最大嗰塊（= sheet 框），淨低先係各電箱板；否則 in-bounds 檢查永遠真。**自己寫 DXF 板偵測時切記排除 sheet 框**。
19. **⚠️ DXF 文字樣式 font 只接受檔名**：`doc.styles.add("SIM", font=...)` 嘅 `font` 屬性只收字型**檔名**（如 `simhei.ttf`），**唔可以俾完整 Windows 路徑**。俾完整路徑 → AutoCAD 搵唔到字型 → 自動替換另一隻字型 → 字形 metrics 改變 → 文字可能出框/錯位（修正：`font_name = os.path.basename(font)`）。精確字形量度如需完整路徑，另存 `FONT_TTF_PATH[0]` 供 verify 用。
20. **⚠️ 字型大小有意妥協（1:1 幾何 vs Word 實際 pt）**：CAD 輸出 `TEXT_H=0.30cm` 係**固定常量**，唔再由 Word `run.font.size`（pt→cm）推算。原因：Word pt 字高轉 cm 後偏細、實體標籤難讀；加工場要「每格清清楚楚」、統一字高比跟 Word 原 pt 重要。**幾何（欄寬/行高/對齊/位置）100% 跟 Word，唯獨字高統一 0.30cm**。若日後要用 Word 原 pt，改令 `TEXT_H` 由 `font_pt` 推算並重跑 verify。
21. **⚠️ RCD 合併 flag 改名**：舊 `--merge-rcd`（開啟合併）已改成**默認自動合併**；新 flag 係 `--no-merge-rcd`（關閉）。手動落指令時**唔好再傳 `--merge-rcd`**（會變 unrecognized argument 報錯）。`regen_label_cad.py` 已唔使傳任何 RCD flag。

---

## 10. 已知卡点（环境限制，照现状处理）

- **DWG→PDF 自动转档**：accoreconsole PLOT（prompt 乱码 + 纸张名格式敏感）、AutoCAD COM（gen_py cache bug）、PowerShell COM（被安全策略挡）三种方法都有环境阻碍。现状方案：优先检查资料夹有无「落实版 PDF」，有就直接用 fitz 提取文字核对；无则告知用户需人手转。
- **卡点不阻断标签牌制作**：标签牌只需回路用途 + 编号（已能从 DWG 解析），PDF 核对只是附加保险。

---

## 11. CAD 向量加工图输出（Word → DXF / SVG 带尺寸标注）【新增能力】

- **目的**：广告商/加工场要 1:1 向量档落料。由 Word 标签牌 docx 读每张 table 的**栏宽（tcW）** + **行高（trHeight）**（单位 cm，python-docx 直接返回准确 cm），1:1 还原几何，再加 dimension 标注（尺寸标柱）。
- **脚本**：`gen_label_cad.py`
  - 必填：`--docx 标签.docx`
  - 输出：`--out 路径.dxf`（默认）或 `--out 路径.svg`；`--svg` 在 DXF 之外再多写一份 sibling `.svg`
  - 可选：`--sheet-w 42`（图框宽 cm，自动取 max(此值, 最宽板+边距)）、`--no-percol`（跳过每栏宽标注，只留总宽/总高）、`--title "..."`、**RCD 宽列自动合并**（默认侦测含「漏電/RCD/RCCB」最左栏即合并为一格高 cell；`--no-merge-rcd` 可关掉）
- **输出格式怎么选（重要）**：
  - **DXF（推荐）**：属于用户所列 `.DWG / .dxf` 家族，AutoCAD / CorelDRAW / Illustrator **均可直接「汇入」**，1:1 保留尺寸与标柱。
  - **SVG**：Illustrator / CorelDRAW 汇入最干净（黑底板 + 白字 + 红色标柱线），适合直接进排版。
  - **.ai / .cdr 原生格式**：属专有封闭格式，脚本**无法直接产生**；但 DXF 与 SVG 都可被 Adobe Illustrator 与 CorelDRAW 直接开启/汇入并保留 1:1 尺寸。若硬要 `.ai`/`.cdr`，在该软件「档案→汇入」DXF/SVG 后「另存」即可。
- **尺寸标柱内容（每张板）**：
  - 总宽：板下水平标注（cm）
  - 每栏宽：板下第二行，对齐各栏（cm）—— 即「规格尺寸标柱」核心
  - 总高：板右垂直标注（cm）
  - 每格保留用途/编号文字（黑底白字）
- **版面**：所有板排进一张图框（A3 宽 42cm 起，自动换行），含图框线 + 标题 + 图例。
- **已验证 1:1 准确**：DXF 量度值 = Word 栏宽之和（例：表1 = 7.2 四P + 6×1.8 一P = 18.0cm），回读 `DIMENSION.get_measurement()` 实测 18.009 / 7.204 / 1.801 cm，与 Word 栏宽一致。

### 11.1 文字必须与 Word 文档「一模一样」（关键铁律）

用户明确要求：DXF/SVG 的**文字排列、位置、大小**要和来源 Word 文档**完全相同**，不能自行估算。

- **字型大小（用戶最終指定）**：字高統一 0.30cm（`TEXT_H=0.30`），唔再用 Word 實際 pt。所有標籤文字（用途/編號/RCD）一律 0.30cm；圖面標題另設 0.6cm。
- **對齊（用戶最終指定）**：多行文字**水平居中 + 垂直置中**（horizontal CENTER、vertical CENTER——整組多行在格內居中、行與行等距上下對齊）。DXF `MIDDLE_CENTER`、SVG `text-anchor="middle"`，x 取 `xs[c]+col_w[c]/2`（格水平中心）。**行距**：行與行間隙 `LINE_GAP=0.18cm`（用戶指定 0.15–0.22 寬松，唔再用倍率），`line_h = TEXT_H + LINE_GAP = 0.48cm`。RCD 合併高格同樣**水平居中 + 垂直置中**。
- **文字框結構（用戶第 5 次確認「唔好一行一字一個框」+ 第 7 次確認「字必須喺框內」）**：
  - **SVG**：每欄 ONE `<text>`（內含多個 `<tspan>` 行），每行列 band 中心 y、`text-anchor="middle"`。`cell_lines` 封頂 2 行。
  - **DXF（關鍵修復）**：**唔再用「整欄一個 MTEXT + 空白行推落 + 擺板垂直中心」**——嗰招喺 AutoCAD 實際渲染時，`line_spacing_factor` / `MIDDLE_CENTER` 解讀差異會令成塊字高出框外。改為**逐 row 一個 MTEXT、垂直置中喺該 row band 內**（同 SVG 邏輯一致）：`cx=xs[c]+col_w[c]/2`、`cy=y_mid_r=(top_r+bot_r)/2`、`attachment_point=5`。每 row 最多 2 行 → 字塊高 ≤2×line_h≈0.96cm ≪ row_h(≥0.9cm)，**絕對出唔到框**。RCD 合併格仍 ONE MTEXT 擺板中心（跨整塊板高、2 行、板高夠高無溢出）。DXF 內文前置 `\pqc;`（段落水平居中）+ `line_spacing_style=2`、`line_spacing_factor=line_h/TEXT_H=1.6`。已驗證：335 白字行，正常行距 0 出界，行距 2x 最壞情況 0 出界；每格 ≤2 行 0 超標。
- **排列 / 行序**：row0 = 用途（上行 3.1cm）、row1 = 编号（下行 0.9cm）。**分界（水平分隔）线必须由顶向下累计**画在 `y = py+ph-Σrow_h[0..r-1]`，即 row0/row1 真正边界，**绝不可**把分界线画在 py+row_h[0]（那会停在顶部 0.9cm 处，与文字错位）。
- **多行字數分配（關鍵新規則，`balance_lines` + `cell_lines`，用戶第 6 次確認）**：**每格（用途 row0 / 編號 row1 各自）最多 2 行**——用戶明言「文字行同線路行嘅段落行數太多，1-2 行就可以」。用戶手改 docx 嘅示例已示範為 1 行，腳本須同步統一。
  - `balance_lines(text)`：n≤2 → 1 行；n≥3 → **2 行**（前半/後半盡量平衡，首行最多多 1 字）：3→2+1、4→2+2、5→3+2、6→3+3、7→4+3、8→4+4、9→5+4。
  - `cell_lines(txt)`：先按顯式 `\n` 分段。**若已有 ≥2 段（作者顯式斷行）→ 保留每段 1 行、唔再拆、整格封頂 2 行（取前 2 段）**；若只有 1 段，過長先過 `balance_lines`（封頂 2 行），非 CJK 原樣 1 行。
  - 關鍵修正：舊版 `balance_lines` 會把單 CJK 段拆成 3–4 行，令長名爆到 3 行；且會把顯式斷行再拆，變成 3 行。新版封頂 2 行後全部格 ≤2。
- **RCD 列自动合并（默认自动）**：docx 的 RCD 宽列（c0）在 row0 与 row1 若都写了「漏電斷路掣\nR C D」（非合并单元格），**默认脚本自动侦测含「漏電/RCD/RCCB」最左栏并合并**为**一格跨整块板高**的高 cell，文字取 row0 一次、按 `\n` 分两段**水平居中 + 垂直置中**，**分隔线不穿过该列**（只从 RCD 右界画到板右）。唔使再手打 flag；要关掉用 `--no-merge-rcd`。
  - **RCD 段换行规则（`rcd_lines` 专属，≠ `balance_lines`）**：CJK 段「漏電斷路掣」若 `len×TEXT_H ≤ 列宽−2·LEFT_MARGIN`（**一行摆得落**）就**整段一行**；摆唔落才拆成**三行**（按余数 1/2/3 分配字数）。

### 11.2 生成後必跑驗證（鎖死「字必須喺框內」+ 每格 ≤2 行）

用 `verify_label_cad.py`（座標解析，唔靠肉眼），唔好只靠 preview：

```bash
PY="<CAD_PYTHON>"

# 重新生成（glob 解析路徑，避開字符坑；RCD 欄自動合併，唔使 flag）
$PY scripts/regen_label_cad.py --glob "<DESKTOP_DIR>/*電箱*/*項目*.docx"

# 驗證：每格 ≤2 行 + DXF/SVG 全部文字入框（含行距 2x 最壞情況）
$PY scripts/verify_label_cad.py --glob "<DESKTOP_DIR>/*電箱*/*項目*.docx"
#   預期輸出：✅ 全部格 ≤2 行 / ✅ 全部入框（含最壞情況）/ ✅ 全部通過
#
# 自動化更徹底嘅「無頭自檢」（ezdxf→PNG 自渲染 + 7 項幾何自檢，唔使等用戶截圖）：
$PY scripts/render_check_label_cad.py --glob "<DESKTOP_DIR>/*電箱*/*項目*.dxf" --png <TEMP_DIR>/check.png
#   自檢項目：[2]入框(含2x行距) / [4]水平居中 / [5]字同字重疊 / [6]板同板/標柱相撞 / [7]RCD行數
#   全部 ✅ → 「自檢全部通過（唔使等用戶截圖）」
#
# DXF → DWG 交接地（生成 .scr，可 --console 無頭轉檔出 .dwg）
$PY scripts/dx2dwg_scr.py --glob "<DESKTOP_DIR>/*電箱*/*項目*.dxf"
#
# regression gate（pytest，自製 fixture 唔靠用戶檔）
$PY -m pytest tests/ -q
```

若報出界 → 第一時間檢查 DXF 文字定位係咪誤用咗「整欄一個 MTEXT + 空白行 + 板中心」舊招（見 §9 第 16 條）。若報「唔居中 / 重疊 / 相撞」→ 查 `read_tables` 欄寬解析或 `layout` 排版 gap 參數（見 §9 第 18、19 條）。

---

## 12. 关键参数速查

- 标签牌：黑底 `000000` / 白字 / 白边框 0.5pt / 字型 SimHei
- 行高：上行用途 1758 twips（3.1cm）/ 下行编号 510 twips（0.9cm）
- 栏宽：MCB 1P=1021 twips（1.8cm）；RCD=1021×P（2P=3.6cm、4P=7.2cm）；馈线 3P=5.4cm
- 纸张：回路 >22 用 A3 横向；边距 1.27cm
- 大标签：80×50mm=4536×2835 twips；编号+「配电箱」48pt 粗体；总开关 64pt
- CAD 图框：默认宽 42cm（A3），高自动；标注单位 cm
- 执行环境：`<CAD_PYTHON>`（ezdxf + python-docx + PyMuPDF）

---

## 13. 成熟度守則（agent 行為）

### 13.1 由「病徵」重診斷，唔好靠估（座標解析 > 肉眼/截圖）
- 用戶貼圖話「字出界」，**唔好第一時間當係斷詞/排版問題**。先問清楚現象（「係字飛出框？定係位置錯？」）。
- 系統唔支援直接睇圖時，**靠 parse DXF/SVG 座標診斷**：逐一比對 MTEXT insert 座標 vs 板框 LWPOLYLINE、計算行中心有冇超出 `[y0,y1]`。本任務就係咁發現「整欄一個 MTEXT + 板中心」喺 AutoCAD 實際渲染會頂出框。
- 一旦鎖定根因，做**可量化**嘅修復（逐 row MTEXT + 垂直置中 row band），再用 `verify_label_cad.py` / `render_check_label_cad.py` 座標自檢鎖死。

### 13.2 確認即沉淀（唔好等「最後」先寫）
- 用戶確認「全完正確」當下，就應該順手寫入知識庫（對應文檔 / README / SKILL / MEMORY），而唔好等到會話尾先一次過補。確認 = 當下可沉淀。
- 每次改咗腳本/參數/採坑，**同一回合內**就更新對應文檔同記憶；斷層會令下次又重蹈覆轍。

### 13.3 自動化優先（唔使等用戶出手）
- 凡係「要人手貼圖確認」嘅步驟，問自己：可唔可以變成腳本自檢？`render_check_label_cad.py`（ezdxf→PNG 自渲染 + 幾何自檢）就係為咗令 agent 唔使等用戶截圖就知有冇出框/唔居中/重疊/相撞。
- 生成器改動後，**先跑 regression gate**（`python -m pytest tests/ -q`）再交付；唔好淨係手動 preview。

### 13.4 路徑/字型等「睇唔到嘅坑」要主動鎖死
- 中文/罕字路徑坑 → 一律 `--glob` 喺 Python 內部解析。
- DXF style `font` 只收檔名 → 用 `os.path.basename()`。
- verify 板偵測要剔走 sheet 外框 → 否則 in-bounds 假 pass。
- 呢啲都係「肉眼查唔到、只喺 AutoCAD 實際開檔才爆」嘅坑，寫入 §9 對應條目，下次直接避開。
