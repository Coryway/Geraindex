# skill 08 · 琴体配置（spec_dict）

> 定位：**给全榜提供唯一可信的规格来源**。不是"再抓一遍网页"，是
> 「三源合并 → 归一化 → 字段级仲裁 → 缺口显式」。
>
> 消费方：① 详情页展示 ② 筛选/对照 ③ **手感（feel）重建**（独立一轮，需用户拍板口径）

---

## v0.2 状态（2026-11-16）

- `build_feel_profile.py` 已改**物理口径**（不再是百分位）：弦长² 张力、1F 毫米线性、指板 9.5–12″ 区间满分、品丝序数 3–6 满分
- 新增 **G6 合成对账**：四轴 → feel 与主榜 `scores.feel` 逐款比对，容差 5e-4（当前 1e-4，95/95 通过）
- 四轴 → 手感分**不在本 skill 实现**，调用 `skills/12_feel_compose/compose_feel.py` 的 `compose_axes()`
- `export_feel_js.py` 已废弃移出（产出的 `feel_profile.js` 已删除，前端改为读 `data.js` 的 `feelAxes`）

---

## 0 用户指示（2026-11-14）

> 把 95 款规格按官方 spec 抓对。之前有试过抓官方，不一定抓的全，可以抓淘宝详细页的配置参数代替，
> 并且不同厂给出的规格参数不一定会对齐。

三条约束：**官方抓不全 → 淘宝兜底；各厂口径不一致 → 必须归一化；先过 DS 再动手。**

DS 预审（2026-11-14）结论：**有条件通过**，六条裁定已内化进本文件（见 §7）。

---

## 1 为什么必须做

`_过目/229` 已确认：全榜 95 款 `feel = 2.629 + 0.999 × tone`，`R² = 0.8814`。
`M = 0.35·tone + 0.35·feel + …` 实际是 `0.70·tone + …`——**手感维度没有独立信息**。

要修 feel，必须先有可信规格。而现有规格的"手感三要素"几乎全缺：

| 字段 | 覆盖率（124 款） |
|---|---:|
| `neck_contour`（琴颈形状） | **5%**（6 款） |
| `fret_size`（品丝规格） | **5%**（6 款） |
| `fretboard_radius`（指板弧度，两个同义键并存） | 33%（41 款） |

---

## 2 字段字典（P-1 冻结件）

### 2.1 分级

- **A 级**：手感模型必需。缺一个该款就不能进"可比榜"
- **B 级**：展示 / 筛选 / 详情页对照
- **C 级**：追溯、型号匹配、辅助核对

> **A 级 = 六项**（2026-11-15 变更）：
> `nut_width` / `scale_length` / `fret_count` / `fret_size` / `fingerboard_radius` / `neck_contour`
>
> `fret_material` 原为 A 级第七项，已降为 **B 级**。依据：全行业不标注该字段
> （主榜 95 款缺 76，覆盖率 20%），Fender 只写 `Medium Jumbo`、Gibson 只写 `Cryogenic Treated`、
> Ibanez 仅高端款写 `Stainless Steel`；且它是**无优劣的取向型字段**（不锈钢＝亮/滑/耐磨，
> 镍银＝暖/咬弦），不能作为"越高越好"的评分输入。手感主导项是 `fret_size`（规格），不是材质。
> 降级后「六项核心全齐」13 → **59** 款（62%）。字段本身仍保留、仍可补，只是不再卡准入。
>
> **`neck_contour` 的判定为「形状名 ∪ 厚度」**（2026-11-15 补充）：
> `neck_contour` 有值 **或** `contour_dims` 有值，即视为该项齐备。
> 依据：形状名只是厚度的粗粒度标签，厚度才是物理量。Yamaha 全系只给
> `20.9→22.9mm @1F/12F` 不给名字（8 款），其证据强度高于只给 `Modern C` 而无厚度的 52 款。
> 改判标准后 59 → **67** 款。真正该卡的只有「两者都无」的 16 款。

| 字段 | 级 | 单位/值域 | 归一化规则 |
|---|:--:|---|---|
| `nut_width` | A | mm 浮点 | 英寸×25.4；剥掉混入的材质词；多值（0 品/22 品）取 0 品 |
| `scale_length` | A | 英寸浮点 | `648mm`→25.51；`24 3/4`→24.75；`628mm`→24.72 |
| `fret_count` | A | 整数 | 从 `frets` 拆出 |
| `fret_size` | A | 枚举，见 §2.2 | 从 `frets` 拆出，**不含材质** |
| `fingerboard_radius` | A | mm 浮点 + `compound` 标记 | 英寸×25.4；`400mmR`→400；复合弧度记 `{lo,hi,compound:true}` |
| `neck_contour` | A | `{vendor_raw, shape_family, thick, nut_w, thick_1st, thick_12th}` | 见 §2.3，**不许只留 thick** |
| `fret_material` | **B** | 枚举 `{nickel_silver, stainless, nickel_silver_us, unknown}` | 从 `frets` 拆出。**2026-11-15 由 A 降 B**，原因见上方说明 |
| `fingerboard` | B | 枚举 `{rosewood, maple, ebony, laurel, baked_maple, other}` | 中英双向映射 |
| `body_wood` | B | 枚举 | 中英双向 |
| `neck_wood` | B | 枚举 | 中英双向 |
| `top_wood` | B | 枚举或 null | 中英双向 |
| `pickup_type` | B | 字符串 + 结构码 `{SSS,HSS,HH,HSH,P90,...}` | 沿用 `_c4_spec.py` 词表 |
| `bridge` | B | 字符串 | 不归一化，保留原文 |
| `tuners` | B | 字符串 | 不归一化 |
| `inlays` | B | 字符串 | 不归一化 |
| `weight_kg` | B | 浮点 | g→kg；`8 lbs`→3.63 |
| `body_finish` | B | 字符串 | 不归一化 |
| `model_no` | C | 字符串 | 保留原厂型号 |
| `origin` | C | 枚举 `{JP,US,MX,ID,KR,CN,BR,...}` | 国名/产地词映射 |
| `year_range` | C | 字符串 | — |

### 2.2 `fret_size` 枚举

`{vintage, vintage_tall, medium, medium_jumbo, jumbo, narrow_tall, extra_jumbo, unknown}`

**不含 `stainless`**——材质归 `fret_material`。历史值 `stainless_jumbo` 必须拆成
`fret_size=jumbo` + `fret_material=stainless`（DS 裁定 ③）。

### 2.3 `neck_contour` 结构（DS 裁定 ②）

```json
{
  "vendor_raw": "\"U\" Shape",
  "shape_family": "U",
  "thick": 0.85,
  "nut_w": 41.02,
  "thick_1st": 21.5,
  "thick_12th": 23.0,
  "thick_seed_src": "mapping_table_v1"
}
```

- `shape_family` 枚举：`{U, soft_V, C, modern_C, deep_C, D, slim_taper, wizard, oval_C, pattern_regular, pattern_thin, wide_thin, baseball, unknown}`
- `thick ∈ [-1,1]` **只是派生弱特征**，映射表版本化（`mapping_table_v1`），不得单独作为 A 级证据
- 缺分量时 `thick` 只进敏感性分析，不进正式模型

---

## 3 数据源与置信度

### 3.1 置信度枚举（DS 新增）

| 值 | 含义 | 能否进 A 级 |
|---|---|:--:|
| `verified` | 官方单品页 / 厂家 spec PDF | ✅ |
| `observed` | 淘宝详情页参数区实测（带快照） | ✅ |
| `inferred` | 从原文可确定性换算（`648mm`→25.51"） | ✅（带换算记录） |
| `family_inferred` | 官方多页逐字一致 + 按**可读判据**归族后取族值（见 §3.4） | ⚠️ 条件准入 |
| `proxy` | 同系列/同型借用（无判别门禁） | ❌ 仅探索 |
| `conflict` | 多源不一致、未裁定 | ❌ |

`family_inferred` 与 `proxy` 的分界在于**有没有判别门禁**：
`proxy` 是「型号名字像 → 借用」，`family_inferred` 是「用页面上独立可读的量（品数、琴枕材质、
半径形态）先把琴唯一归入某族，再取该族内 100% 一致的字段值」。前者可被任意叙事带偏，后者可被反例证伪。

### 3.2 字段级仲裁矩阵（DS 裁定 ①：不是全局排序）

| 字段 | official | taobao | findmyguitar |
|---|:--:|:--:|:--:|
| `nut_width` | 1 | 2 | **3（不得单独赢）** |
| `scale_length` | 1 | 2 | **3（不得单独赢）** |
| `fret_count` | 1 | 2 | 2 |
| `fret_size` | 1 | 2 | **3（不得单独赢）** |
| `fingerboard_radius` | 1 | 2 | **3（不得单独赢）** |
| `neck_contour` | 1 | 2 | **3（不得单独赢）** |
| B 级字段 | 1 | 2 | 2 |

数字为优先级（1 最高）。**「不得单独赢」= 当 official 与 taobao 都缺、只剩 findmyguitar 时，
该字段记为 `conflict` 并进人工队列，不许直接采信。**

官方内部再分级：**单品页 > 厂家 spec PDF > 系列页 > 经销商页**。系列页不得等同于单品实测。

### 3.3 缺失原因码（DS 新增）

`official_no_page` / `official_page_no_field` / `taobao_no_param_tab` / `taobao_no_field`
/ `third_party_only` / `source_conflict` / `manual_pending` / `not_applicable`
/ `family_gate_failed`（有官方页但判据不足，无法唯一归族）

### 3.4 Keipro 三族实证（2026-11-14）

> ⚠️ **适用范围：仅 Keipro 新世代（官网在售，2024–2026）。**
> 主榜那批旧代号（KS100/KS150/KS170/KS200/KS350/KS470/KS650/KT100/KT150/KT200）属**上一代**，
> 官方参数表写明琴颈为「**亚洲C型**」、琴枕为「**42mm 牛骨**」，与下表新世代的
> `C Slim` / `POLYMERS` 不同代。**族级值不得跨代套用**（见 §3.4.1）。
> 旧世代参数表存于 `原始数据/mb/_tm_spec_imgs/keipro_matrix.jpg`（含 top/bot 分图），
> 但它**没有指板半径字段**，且无单品 item 归属（一页多琴），故补不了 radius/contour_dims。

官网 `keiproguitars.com`（WooCommerce）每个产品页都有完整 NECK 段，**含琴颈厚度**。
14 个页面抓取归集为 3 个族，族内六项逐字一致：

| 族 | 系列 | 品数 | 半径 | 琴枕宽 | 琴枕材质 | 琴颈形状 | 页数 | 价带(USD) |
|---|---|:--:|:--:|:--:|---|---|:--:|---|
| **T** 传统 | Classic / Standard / Deluxe / Deluxe Pro / Custom | 22 | 9.5″=241.3 | 1.65″=41.91 | POLYMERS | C Slim 0.8→0.85″ | 10 | 335–825 |
| **M** 现代 | Modern / Modern-M / Modern Pro-Q | 24 | 12″=304.8 | 1.65″=41.91 | BONE | C Slim 0.8→0.85″ | 3 | 585–825 |
| **X** 重型 | X-Metal T6-F-Ash | — | 12″~16″ 复合 | 1.69″=42.93 | GRAPHITE | C Slim 0.8→0.85″ | 1 | 1585 |

跨族不变量（14/14 页）：`neck_contour = C Slim`、`scale = 25.5″=647.7`、
`fret_size = Medium`、`fret_material = Nickel`、`contour_dims = 20.32 / 21.59 / taper 1.27`。

**归族门禁**（必须是页面上独立可读的量，禁止靠型号名相似）：
- `fret_count == 22` → T（M/X 均为 24 品，故 22 品唯一指向 T）
- `fret_count == 24` 且琴枕材质明确为牛骨 → M（X 族为石墨琴枕）
- 半径呈复合 `12″~16″` → X
- 其余一律 `family_gate_failed`，留空

**已知书写变体（解析器必须容忍）**：
`C SLIM 0.8" ~0.85"` / `C SLIM 0.8"~0.85"` / `C SLIM 0.8"-0.85"` / `C SLim 0.8"` / 仅 `C SLIM 0.8"`（单值，只给 1F，**不得补 t12**）；
琴枕宽度存在 `1,65"` 逗号小数（=41.91mm，若误当千分位会读成 165″）。

**厚度参考点未标注**：官网页从不写 `1F`/`12F` 字样，`0.8"~0.85"` 取 1F/12F 是行业惯例推定，
落盘必须带 `ref_label: unlabeled_by_vendor` + `ref_assumed: 1F/12F`，不得当作已标注。

产物：`skills/08_spec/keipro_official.py`（14 页原文快照 + 13 条一致性断言）
→ `原始数据/mb/_keipro_official_20261114.json`。

### 3.4.1 族级推断的前置条件：先证同代（2026-11-14 教训）

`family_inferred` 的隐含前提是「同族即同规格」，而这个前提**必须先证明两件事**：

1. **同代**——型号代号体系一致。用户侧在售型号若用的是旧代号（KS150），官网若已改成系列名
   （KS-Classic），那不是改名，可能是换代。**必须找旧世代的官方/半官方参数表对照**，
   比对琴颈名、琴枕材质这类高辨识字段；出现「亚洲C型 vs C Slim」「牛骨 vs 聚合物」即判定跨代。
2. **族判别量确实存在且独立**——品数、琴枕材质这类页面独立可读的量，不能是同一句宣传词的复述。

跨代一旦成立，**整族取值对旧代号全部作废**，不能只作废「看起来不同」的那几项。

反面案例：本次把新世代 T 族（C Slim / 9.5″ / POLYMERS）套到旧代号 6 款上，
产出 35 个字段单元，全部回滚。产物留档 `_v4_deprecated_跨代误套.json`。

### 3.4.2 已知未解冲突

| 型号 | 冲突 | 双方 | 处置 |
|---|---|---|---|
| `Keipro|KT100`、`Keipro|KT150` | 品数 21 vs 22 | 淘宝详情页（有 item id，observed）vs 官方参数表翻拍（无 item） | **未裁定，保持原值 21**，不采信翻拍表 |

---

## 4 输出契约

`原始数据/mb/_spec_dict_<YYYYMMDD>.json`

```json
{
  "_meta": {"ver":"spec-1.0.0","generatedAt":"...","sources":{...},
            "fieldDict":"P-1 frozen 2026-11-14","arbitration":"matrix v1"},
  "keys": {
    "<琴型 key>": {
      "nut_width": {"raw":"1.615\" (41.02 mm)","norm":41.02,"unit":"mm",
                    "source":"fender_official","confidence":"verified",
                    "url":"...","retrieved_at":"2026-11-14T15:20:11+08:00",
                    "snapshot_hash":"sha256:...","conflict":[]},
      ...
    }
  },
  "gaps": {"<key>": {"neck_contour":"official_page_no_field",
                     "fret_size":"third_party_only"}},
  "arbitration_log": [{"key":"...","field":"nut_width",
                       "candidates":[{"source":"taobao","norm":42.0,"conf":0.9},
                                     {"source":"findmyguitar","norm":42.86,"conf":0.6}],
                       "winner":"taobao","rule":"matrix_v1:A=1>2>3"}]
}
```

**每个值必须四件套齐全**：`raw` / `norm` / `source` / `confidence`；
淘宝值另加 `source_item` + `source_shop` + `snapshot_hash`。

---

## 5 契约 A1–A12

| # | 断言 |
|---|---|
| A1 | 每字段必有 `raw` + `norm` + `source` + `confidence`，缺一即失败 |
| A2 | `norm` 与 `raw` 数值自洽（±0.5%），否则判 `convert_error` 并列出 |
| A3 | 冲突必须进 `conflict[]` + `arbitration_log`，不许静默取一个 |
| A4 | `gaps` 与主榜 95 款 A 级字段缺失**逐条一致**（双向：不许漏报、不许虚报） |
| A5 | 同义键（`fretboard_radius` / `fingerboard_radius`）合并后不许两处残留 |
| A6 | `frets` 拆分后 `fret_count + fret_size + fret_material` 三者可逆还原原串（`_legacy_frets` 仅审计，不算迁移完成） |
| A7 | 每个 `taobao_*` 值必须带 `source_item` + `source_shop` + `snapshot_hash` + `retrieved_at` |
| A8 | `confidence ∈ {verified, observed, inferred, proxy, conflict}` |
| A9 | A 级字段的 `confidence` **只能是 verified / observed / inferred**；出现 `proxy`/`conflict` 必须同时进 `gaps` |
| A10 | `proxy` 值不得出现在任何 A 级字段的 `norm` 中（只许进 `candidates`） |
| A11 | 全仓无明文密钥（沿用 A12 扫描口径） |
| A12 | 主榜 95 款每款都有 `_spec_dict` 条目（可以全字段缺失，但键必须在） |

---

## 6 执行计划（DS 修正后）

| 阶段 | 内容 | 门禁 |
|---|---|---|
| **P-1** | 冻结字段字典 / 仲裁矩阵 / 置信度 / 缺失码 / 契约定稿；20 款黄金集双人标注；**feel 效度验收预注册** | 本文件 + `_golden20.json` |
| **P0a** | 解析器 + 测试集（`nut_width` / `scale_length` / `frets` 三组，每组 ≥10 正例 + 5 反例） | 测试全过 |
| **P0b** | `frets` 双写迁移（**不改原值**，新增 `fret_count/size/material`，可回滚） | A6 可逆 |
| **P0c** | 同义键合并（两个 radius） | A5 |
| **P0d** | 冲突清单（现有 124 款内部矛盾） | A3 |
| **P1** | 官网扩到 95 款 | `official_miss` 清单 |
| **P1b** | 淘宝小批 10–20 款 A/B：命中率 / **准确率 ≥90%** / 风控失败率 | 不达门槛不扩量 |
| **P2** | 淘宝扩量（仅对 `official_miss`） | 失败可解释 + 快照可复现 |
| **P3** | 字段级仲裁 + 缺口 | A4 双向一致 |
| **P4** | 抽检 20–30 款（必须含冲突款 / 缺失款），算字段准确率与仲裁一致率 | 不通过回炉 |
| **P5a** | feel 效度预注册与敏感性 | 未达门槛不替换旧 feel |
| **P5b** | feel 重建 | 用户拍板 |

**P5 不并入本轮交付。**

---

## 7 DS 预审裁定留档（2026-11-14）

1. 三源冲突 → **改**：字段级 + 置信度级；A 级中 findmyguitar 不得单独赢；两方冲突进人工队列
2. 同系列代理 → **否**：`proxy` 不进 A、不参与仲裁；「假齐比缺口更坏」
3. `neck_contour` 单一 `thick` → **不够**：保留 `vendor_raw + shape_family` + 分量
4. `frets` 拆分只留 legacy → **不够**：三字段 + 可逆解析 + 迁移测试
5. 淘宝点 tab → **有条件采纳**：先小批验准确率与风控，<90% 不扩量，要 `snapshot_hash`
6. 任意 2 项代理重建 feel → **否**：留空 + 标不可比；插补需预注册缺失模型

**DS 指出的最大风险**：把"规格可得性"当成"测量效度"。
`规格 → 手感` 的映射可能同样无效，做完只是把「tone 平移」换成「规格平移」。
故 P5a 必须先预注册验收门槛（feel~tone 的 R² 上限、残差区分度、与盲评相关性、排名稳定性），
达不到就不替换旧 feel。

**次风险**：缺失非随机。官网 40 / 淘宝 10 / 第三方 24 款，补不齐的款可能集中在特定品牌与价位，
A 级留空后正式榜只剩资料全的大牌 → 性价比结论系统性偏向。`gaps` 必须按品牌/价位统计披露。
