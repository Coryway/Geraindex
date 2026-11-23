# skill 02 · 闲鱼二手价（E）— v1.1（2026-11-11 封盘）

> 2026-11-08 · 按 `_过目/归档/148_skill封装规范` 七段 + 双链路封装
> **2026-11-10 首版封板**：P0–P5 全通；视觉闸（0g）全量落地并写入主榜。
> **2026-11-11 v1.1 终版封盘**：Q1 封顶双源取小、Q4 子型号词表、子品牌中文名跨品牌拦截；
> 契约基线与自检基线按新判据重冻（**18 道闸 / 116 款 / 有 E 96**）。
> **采集内核不重写**：`抓取脚本/mb_b_xy.py`（648 行）+ 增量器 `_xy_re_20261102.py`；
> **判据不重写**：`e_channel.py` + `_pool_e_merge_20261029.py::merge_one`。
> 本 skill 只负责：映射（P0）、入口（`collect.py`）、账层（`run.py`）、契约（`contract.py`）、自检（`selftest.py`）。
>
> **本 skill 不写主榜**。唯一写入口是 skill 07。`_recalc_e_batch --write` 永久禁用。

```
skills/02_xy_price/
├── SKILL.md        七段契约（本文件）
├── run.py          E patch + 四层账（--layers --all / --keys / --pool）
├── contract.py     A1–A6（--freeze 冻基线）
├── collect.py      选款 + 调 _xy_re 内核 + 去重入池（--mode=incr|new）
├── vision.py       识图闸 0g 前置：候选取图 → 2×2 拼板 → manifest（判读交多模态模型）
└── _skill2_baseline.json   闸 kill rate 冻结基线（17 道，2026-11-10 重冻）
    _selftest_baseline.json 逐款 E 冻结基线（115 款 / 有E 96）
```

---

## 1 输入契约

| 输入 | 路径 | 关键字段 | 缺失怎么办 |
|---|---|---|---|
| 闲鱼池 | `原始数据/mb/mb_pool_xy.json` | `key/candidates[]/scrapedAt/excludeTokens` | 文件缺 **exit 2**；单款缺 `no_pool_entry` |
| 映射表 | `原始数据/mb/_xy_keymap.json` | `board_to_pool / pool_to_board / missing / negative` | 缺 **exit 2**（先跑映射重建） |
| 采集配置 | `抓取脚本/mb_models.json` | `models[].{name,pool,xy,tokens,deny,allow}` | 款不在配置 **exit 2**（`new` 走登记） |
| 主榜（只读） | `MultiBrand_Ranking_Data.json` | `key/model/price.usedMedianCNY/isNewModel` | 缺 → 只出 patch 不比对 |
| ref(一手参考价) | `build_specs_ref → build_models_ref → build_mainP → ref_override` | 同 `_pool_e_merge` | 无 ref → 价格带闸关闭 + `ref_missing` |
| S(海外二手价) 兜底 | `mb_pool_p_os_20261102.json` / `mb_pool_rev.json` | `median_cny` | 缺 → 不兜底，记 `no_s` |
| 全局锁 | `原始数据/mb/net_lock.json` | `owner/skill/pid/since` | 缺失视为空闲；`owner=taobao` → **exit 3** |

**映射铁律**：`alias → MAIN2POOL → norm(去「线 |」前缀/去空白/小写) 全等`。**禁止** `split("|")[-1]` 猜、禁止模糊匹配（E25）。
P0 实测（2026-11-11 终态）：主榜 116 款 → **116 全可映射**、`missing=0`、池内 **24 key** `out_of_board`。
> 2026-11-10 23:2x 修：`Squier|CV Strat`（正式 #19）曾因 11-08 建表时主榜键名是 `Squier|Classic Vibe`，改名后 `missing` 记录陈旧，导致该款**永远排不进补采队列**。已按池内同名 key 补 exact 映射（`patchedWhy` 见映射表 meta），禁止再用「池内真没采」解释。

**时点契约**：每条样本记 `collectedAt`；E 只用「当期快照」，禁止跨月混算。
池样本无该字段时按「款级 `scrapedAt` → 池文件 mtime → 池 meta」回退，标 `ts_inferred: true`。

---

## 2 输出契约

`原始数据/mb/_skill2_e_<YYYYMMDD>.json`（skill 07 的唯一输入）：

```json
{"meta": {"skill":"02_xy_price","version":"1.0","builtAt","pool_file","pool_sha1",
          "pool_keys":141,"n_row":115,"n_gap":25,"n_has_E":112,"n_mapped":115,
          "n_missing":1,"n_out_of_board":24,"rules":"e_channel v3 …"},
 "rows": [{"key","uid","pool_key","pool_keys[]","ref","ref_src",
           "n_cand","n_personal","n_channel","n_total","n_bucket","n_consign",
           "personal_median","channel_median","E_median","E_src","E_basis",
           "E_p25","E_p75","E_span_pct","E_span_state","E_grade",
           "capped_80p","cap_value","channel_share","n_sellers","rank_eligible",
           "drop{…}","brand_unknown","ref_suspect",
           "E_prev","delta_pct","action","verdict"}],
 "gaps": [{"key","reason_code","note"}]}
```

- **`E_median` 是 merge 层原值**（含 `E≤0.8×ref` 守卫，`capped_80p` 留痕）；**不再做 cap 派生**——那是 skill 07 的出口。
- `action` 三档：`silent_adopt`（\|Δ\|≤10%）｜`adopt_flag_recheck`（10–20%）｜`attribution_required`（>20%）。
- 幂等：按日期命名不覆盖历史；同款重跑只更新该 key（`prev.update(new)` 整行替换）。
- 只读：本 skill 任何命令都**不写** `MultiBrand_Ranking_Data.json`。

---

## 3 风控参数（不可放宽）

| 操作 | kind | gap | burst | 熔断 |
|---|---|---|---|---|
| 搜索页 | `xy_search` | **12–25s** | 4 次后 90–150s | 命中滑块/登录墙 → `Halted` + `_xy_halt.flag` |
| 详情页 | `xy_detail` | **12–25s** | 5 次后 150–300s | 同上 |
| 切款暂停 | `xy` 桶 | 12–25s | — | `MODEL_PAUSE_BY_BUCKET` |

- `xy_turbo` **永久禁用**；`mb_b_xy` import 时把档位钉到 `xy_fast` 并断言。
- **并行禁忌**：闲鱼 + 淘宝 **禁止**（共用风控）；闲鱼 + 京东 / B站 / Reverb **可以**。
- 单款上限 `pages=2`、`detail≤10`；所有脚本（含一次性）必须过 `net_governor` 的 `GOV.wait()/mark()`。

---

## 4 三道闸（按 GATES.md 归属，同一道闸不许两层各写一份）

| 层 | 在哪 | 判什么 |
|---|---|---|
| **L0 搜索词**（2026-11-09 新增，成本最低、收益最大） | `mb_pool_xy.build_queries` + `QUERY_HEAD` / `QUERY_ALT` | ① **品牌必须用大众说法**：国外品牌优先英文名（Gretsch/Jackson/Schecter/Eastman），生造音译（格雷奇/斯科特/杰克逊）搜了等于没搜；`mb_models.json` 可设 `queryBrand:""` 关掉中文首查询 ② **圈内俗称必须进首查询**：美超/美专/日芬/墨豪/大G —— 实测搜「Fender Ultra II Tele」全是商家全新，搜「美超二代」立刻出个人二手 ③ **三个词要覆盖三种「说法」，不是一个名字的三种「译法」**（2026-11-16 用户拍板）—— 闲鱼只抓 2 页×30=60 条上限，Q1 抓满后 Q2/Q3 若只是英文/中文写法，返回同一批商品。实证 MIJ Hybrid II Tele：Q1 60 条 → Q2 只多 8 条 → Q3 多 0 条。需要换叫法（「日芬融合2」vs「Hybrid II」）。用 `QUERY_ALT` 配三词，每词 ≤16 字符 ④ **Q1/Q2 管召回，Q3 管精度：Q3 拼「琴型」而不是「电吉他」**（用户原话：「Q3 多加 电吉他 三个字有点浪费…面对 tele st lp sg 琴型，可以在后面加上对应的琴型增加准确率，比如 tagima T805 st」）—— 「电吉他」是泛词，加了只稀释排序；琴型（st/tele/lp/sg/jm）才能把同系列不同琴型分开。**型号名里已含琴型的（SG/Strat/Tele/Jazzmaster）不重复加**，改拼系列名或词序。模板：`tagima T805 st` / `Shijie STE st` / `Eastman SC52 lp` / `PRS Silver Sky st` / `jackson SLX st`；7 弦款用弦数（`Solar A2.7 七弦` / `Solar A2.7 7弦`） ⑤ **中文名不确定时，一律不用中文，改「中英互换」**（用户原话：「不确定的时候不要用中文名，之前也强调过了。如果你不确定的话，中英文互换搜索」）—— 已确认可用：世杰 / 伊斯特曼 / 塔吉玛 / 日芬 / 美超 / 美专 / 墨芬 / 芬达 / 雅马哈 / 依班娜 / 易普锋 / 吉普森 / 法丽达 / 斯奎尔；**禁用**：格雷奇 / 杰克逊 / 斯科特（生造音译，`queryBrand:""` 关掉）。不确定名的款靠「品牌+型号 / 系列全称 / 词序颠倒」拉开三种说法，不靠译名 |
| **L1 采集时** | `mb_b_xy.classify/classify_v2` | 不依赖 ref/样本集的死闸：`FAKE_BRANDS` 仿制牌、品牌 token 不匹配、贝斯排除、纯配件/垃圾（进 `quarantine` 不丢）、求购贴 |
| **L1.5 识图（0g，强制并联）** | `vision.py` 产拼板 → 模型判读 → `collect.py` 执行 | 文本初筛通过后**必须看图**：琴型（Strat/Tele）、琴头 Logo、护板贴纸、指板材质、是否实拍。判决写 `原始数据/xy/vision_2026/verdicts.json`（`{itemId: {model,kind,verdict,why}}`），`verdict != ok`（wrong_model / wrong_gen / not_photo / channel / new / dup）**在入池前剔除并计数**。**2026-11-10 已全量落地**：1,715 条判读、覆盖 100%，kill rate 视觉总 17.3%（`vision`）+ 2.9%（`vision_model`）；细则见 §11 |
| **L2 合并时** | `_pool_e_merge.merge_one` | 全部依赖样本集/ref 的闸：`variant_guard`/`family_guard`/`SIBLING_TOKENS`/`CROSS_MODEL`/`CONFIG_PAIR`/`is_new`/`is_near_new`/`is_junk`/`EXTRA_JUNK`/`SWAP_ONLY`/`WANTED`/**`LIMITED_RE`（2026-11-16 新增：Noir/FSR/石桥/Blanc/限定版等不同配置的特别版剔除；仅换配色的限量不剔；签名款整款走 `LIMITED_SKIP_KEYS` 白名单不套用）**/同标题同价去重/`seller_kind`/价格带/`n_gates`/span |
| **L3 计算时** | skill 07 | `cap` 派生与封顶、`R(保值率)/V(净成本)/O(原始分)/T(指数分)` |

L1 输出必须「全记」（`title/price/full/credit/seller/sold/want/cond`），L2 每处丢弃必须有 `reason_code` 且**计数**。

---

### 4.1 搜索词怎么写（2026-11-16 定稿，配 `QUERY_ALT`）

**一句话**：短、尾巴带型号、三个词三种叫法、不确定不用中文。

| 规则 | 怎么做 | 反例 |
|---|---|---|
| **短** | 每词 ≤16 字符（中文 1 字算 1） | ❌ `Fender American Professional II Stratocaster`（44 字符） |
| **型号放尾部** | 品牌/俗称在前，型号号收尾 | ✅ `tagima T805` / `日芬 Hybrid2 tele` |
| **不堆全名** | 卖家不会写官方全称，写了也搜不到 | ❌ `芬达 MIJ Hybrid II Telecaster`（实测只 9 条，新增 0） |
| **不带「电吉他」** | 泛词，只稀释排序（2026-10-27 实证） | ❌ `雅马哈 RSE20 电吉他`（目标掉出前 60） |
| **Q1/Q2 管召回，Q3 管精度** | Q3 拼琴型：`st / tele / lp / sg / jm` | ✅ `tagima T805 st` |
| **三词三种叫法** | 换称呼，不是换语言 | ✅ `日芬 Hybrid2 tele` / `日芬 融合2 tele` / `Fender Hybrid II Telecaster` |
| **中文不确定就不用** | 靠词序/系列名拉开 | ✅ `Solar A2.7` / `Solar A2.7 七弦` / `Solar A2.7 7弦` |
| **中文名必须实测 n>0 才留** | 「看着对」不算数，先搜一次看返回条数 | ❌ `伊斯特曼 SC52/T59/Juliet` 全 0 条、`塔吉玛 T805` 0 条；✅ 只有 `世杰 STE Pro` 抓满 30/30 |
| **型号名已含琴型的不重复加** | SG/Strat/Tele/JM 再加等于 Q1 | ✅ `Squier CV Strat` / `Squier Classic Vibe Strat` |

**已知有效的琴型后缀**：`st`（Strat 型/superstrat）、`tele`、`lp`（单缺角）、`sg`、`jm`。
**不适用**：空心/半空心（Gretsch G2622T、Eastman T59）、双缺角实心（Eastman Juliet）——
这几种不在四类里，硬套会把别的琴型混进来，**不确定就不加**（用户 21:17 明确："不确定的就别套"）。

**琴型后缀实测边界（2026-11-16 22:00 四批重采实录）**：后缀本身有效，但**在货少的款上会把结果打成 0**——
`PRS Silver Sky st` 30/30 满、`jackson SLX st` 20 条，而 `tagima T805 st` 0 条、`Eastman SC52 lp` 0 条。
所以 0 条**先怀疑"这琴没货"**，别急着怪后缀；Q1/Q2 有结果就能兜底。

**为什么 Q3 不拼「电吉他」**（用户 21:13）：
Q2 已经把该搜的搜到了，Q3 再加"电吉他"是缩范围而不是换说法，等于浪费一个名额。
换成琴型才能把同系列不同琴型分开（T805 是 st，写 `tagima T805 st` 才精确）。

**为什么 Q3 不拼「电吉他」**（用户 21:13）：
Q2 已经把该搜的搜到了，Q3 再加"电吉他"是缩范围而不是换说法，等于浪费一个名额。
换成琴型才能把同系列不同琴型分开（T805 是 st，写 `tagima T805 st` 才精确）。

---

## 5 自校验 A1–A6（落盘前跑，不过 exit 2）

| # | 断言 | 实现要点 |
|---|---|---|
| **A1** | `n_bucket + Σdrop == n_cand` | `n_bucket`＝个人+渠道+寄卖+店用（`merge_one` 新增字段）；`near_new_unknown_kept` 是「保留并计数」，**不算 drop** |
| **A2** | 每行 `pool_key` 在池内、`key` 在映射表；主榜款必须出现在 rows 或 gaps | 缺 → exit 2 |
| **A3** | `capped_80p=True` 必须有 `cap_value + ref`；`E > 0.8×ref` 却未标记 → 失败 | 防止「封顶没生效」 |
| **A4** | `--require-all`：行 + 缺口 == 主榜款数，缺口必须带 `reason_code` | 0 挂牌款不算红 |
| **A5** | `\|Δ\|>20%` 必须 `action=attribution_required` | 防「悄悄换价」 |
| **A6** | 每道闸 kill rate 与 `_skill2_baseline.json` 比，**上升 >5pp → 失败** | 绝对 >20% 只告警：`hard_new` 在闲鱼天然 28%，用绝对阈会每轮判红 |

**实测（2026-11-10 封板轮）**：115 款 / 25 缺口 / **96 有 E** / A1–A6 **0 失败**；**17 道闸**已按新判据重冻（较 11-08 新增 `vision` 17.3%、`vision_model` 2.9% 两道基线）。重冻理由：视觉闸首次全量生效，旧基线里该闸恒为 0%，不重冻会每轮 A6 判红。

---

## 6 已知坑（全部有实证）

| 坑 | 现象/根因 | 修法 |
|---|---|---|
| `sellerWhy` 拼进 blob | 「(无店铺…)」含「店铺」→ 个人一手琴被当渠道剔 | blob 不含 `sellerWhy` |
| `EXTRA_JUNK` 不看否定语境 | 「没有琴箱」被判配件 | 否定前缀守卫 |
| `双单双` 不是 HSH | 拾音器配置误判 | 只认 `HSS/HSH/SSS/P90` |
| 标题插空格 | `cust om shop`、`Firebi rd` 漏拦 | `_nows()` 去空白（E24） |
| `brand_unknown` 放行 | 真闲置常只写型号 | **保留**，禁止加「无品牌+低价」通用闸 |
| 「新品」弱全新词 | 与「自用/95新」同现仍判全新 | 弱词需强信号才判全新 |
| 同条挂单重复 | SG Special 同标题同价 9,580×4 | 同标题同价去重（E12） |
| n=2 两点中点绕过封顶 | E 被后续分支改回高位 | 钳制统一放函数出口（E22）；本 skill 不派生 cap |
| 漏 `apply_pool_patches` | 回归值与线上「同源不同输入」（X Soloist 5,872 vs 3,840） | `run.py` 必过 `apply_pool_patches`（E23） |
| 按批次日取数 | 月度只补少量款 → 其余全「无基准」 | 款级最新采样 + 45 天 `stale` |
| 采集产物不入池 | 「采了不用」已 14 例 | `collect.py` 收尾自动入池（去重 + 备份 + 断言） |
| 诊断脚本另写数据源 | 诊断与生产不同源 → 假差异 | 数据解析收一层，三处共用 |
| 取图 JS 先入为主 | 220px 缩略图先占 `seen` 集合，把同图 1280px 原图挤掉 → 全部取图失败，**看起来像风控** | `xy_item_imgs.js` 先按分辨率筛，再按 `base` 去重（E32） |
| 详情页等待过短 | 只等 5.4s，图片区常未渲染 → 取 0 图、误判「已下架」 | 轮询至 12s，连续 3 条取不到才熔断（E33） |
| 判读不分片 | 单会话连续看上千张图，中断即全丢 | 55 条/片 + 每批 5 张落盘，合并脚本只做「读-改-写」（E34） |
| 视觉闸把纯渠道款打空 | 旧口径允许 channel 降权 cap 0.50，视觉闸 channel 一律剔 → E 从有值变无值 | 用户 2026-11-10 裁定 **A：沿用旧值 + 标 `vision_gap`**，主榜不动（§11.4） |

---

## 7 失败处理（禁止 `continue`）

| reason_code | 触发 | 后续 |
|---|---|---|
| `empty_candidates` | 搜索 0 条挂牌 | 换词重采（记已试 `queries`） |
| `no_price` | 卡片无价 | 剔除并计数 |
| `captcha` / `halted` | 滑块/登录墙 | 停手等人工，**不重试不换档** |
| `mapping_missing` | 池 key 对不上主榜 | 进别名待办，不出 patch（**当前 0 款**；2026-11-10 已清 `Squier\|CV Strat`） |
| `out_of_board` | 池 key 明确不在主榜（24 个） | 保留不重采，显式计数 |
| `config_missing` | 款不在 `mb_models.json` | `incr` exit 2；`new` 走登记 |
| `ref_missing` | 无 ref | 价格带闸关闭，E 降参考级 |
| `stale` | 快照 >45 天 | 进重采队列 |
| `span_very_wide` | `E_span_pct>60%` | 不排名（skill 07 执行） |

---

## 7.9 采集范围：什么采、什么不采（2026-11-16 用户规则）

**规则原文**：「采集脚本每次都跳过，每次维护时都需要包含观察区内的琴（不是手动移出的高价琴），当二手数据仍不足时，再移去观察区。」

| 榜单状态 | 采不采 | 说明 |
|---|---|---|
| **正式**（95） | ✅ 必须 | |
| **观察区**（11） | ✅ 必须 | 因数据不足下去的，仍要采 —— 样本补够了才有机会回正式榜 |
| **候选款（未入榜）** | ✅ 采 | 采都采不到就永远没有评估机会。2026-11-16 恢复 8 款：Ibanez AZ/RG Prestige、Gibson Explorer '80s、Shijie TLV-P90S、Tagima TG-530 PRO / CS-3、Keipro KT/KS vintage。用户原话：「恢复，反正闲鱼搜不齐的就不要」——**采回来评估，样本凑不齐就不入榜** |
| 出榜 · 高价（R18 一手价 > 20000） | ❌ `pool:false` | 6 款：Ultra II Strat/Tele、AV II '61 Strat / '63 Tele、LP Standard '50s、PRS S2 Custom 24 |
| 出榜 · 用户裁定剔除 | ❌ `pool:false` | Eastman Romeo（官方零售）、PRS SE NF3（国内无有效样本） |
| 出榜 · **签名琴** | ❌ `pool:false` | Farida JC-1：二手价与一手价接近（E/P = 0.80，一手 12,800 只降 20%），**保值率口径对签名琴失真** —— 收藏属性让二手价被人为撑住，按「净亏损」排名会系统性抬高它们 |
| 出榜 · 保留观察 | ✅ 继续采 | Ibanez AZ224F（出榜原因未写明，用户 2026-11-16 裁定保持采集，数据留观察） |

**签名琴要单独判断**：普通量产琴二手价随年份稳定下跌，保值率能反映「亏多少」；签名款/限量款的买家是粉丝，二手价常被人为抬高甚至接近全新，用它算净亏损会得出「这把琴几乎不亏钱」的假象。遇到签名款先看一眼 `E/P`：**> 0.75 就要怀疑**，不要直接进榜排名。

- 停采必须同时写 `poolReason` 字段，说明为什么停 —— 否则下次没人知道该不该恢复。
- **硬闸两条**（`skills/00_start/preflight.py`）：
  - 「采集范围」：在榜却停采 → **BLOCKED**；在榜但找不到配置 → WARN
  - 「停采合规」：**停采只允许出现在出榜款上**，候选款被停采 → WARN（等于永不评估；2026-11-16 就是靠这条发现 8 款被静默冻结）

### 数据不足怎么处理（不是一刀切移走）

```bash
PY="python"
"$PY" skills/02_xy_price/e_gap_sweep.py                  # 只看
"$PY" skills/02_xy_price/e_gap_sweep.py --apply          # 记账 eGapStreak
"$PY" skills/02_xy_price/e_gap_sweep.py --apply --move-out   # 连续 2 期才移观察区
```

判定分两档，**不能混**：

| 情况 | 处理 |
|---|---|
| `E` **算不出来**（n=0/1，过闸样本不足） | 记 `eGapStreak + 1`；连续 **2 期**才移观察区 |
| `n < 3` 但**有 E**（两点中点） | 不算不足，只标「不参与排名」（`rank_eligible=false`） |

为什么不能把 n=2 也算不足：2026-11-16 实测有 6 款属于这一档（Charvel DK24、Eastman Juliet/T59、MIJ Trad 60s Strat、PRS NF53、Shijie PD-Pegasus），一刀切会在两期后把它们全移走，而它们其实有可见的参考价。

### 本轮修的三处（都是同一族：配置与榜单脱节）

1. **正式榜款被关采集**：Epiphone LP Special DC、Shijie PD-Pegasus 是 `pool:false`，二手价永久冻结 → 已恢复。
2. **配置名与主榜 key 不一致**：配置叫 `Contemporary Jazzmaster HH`，主榜 key 是 `Contemporary JM HH` → 采集永远 0 条，已改名对齐。
3. **出榜琴还在白采**：8 款（6 高价 + 2 裁定剔除）每次全量都采一遍，白花 15 分钟 → 已停采。

---

## 8 命令

### 8.0 起浏览器（2026-11-16 实测定稿，别再猜）

**铁律：浏览器和采集必须在同一次工具调用里串联。**

```bash
cd "<项目根>"
bash 抓取脚本/edge_up.sh && \
PY="python" && \
"$PY" 抓取脚本/mb_pool_xy.py --keys-file 原始数据/mb/_rekeys_xxx.json
```

| 做法 | 结果 | 实证 |
|---|---|---|
| **shell 先起 Edge → 再跑采集**（上面这条） | ✅ 端口 1s 就绪，采集正常 | 2026-11-16 单款 98s、3 款 312s 全通 |
| 让用户双击 bat | ⚠️ 能用但没必要，用户经常找不到文件 | 用户两次问「bat 在哪」 |
| 让 python 自己 `Popen` 起 GUI Edge | ❌ 进程被 SIGTERM，脚本**无任何输出** `exit 1` | 2026-11-16 复现 2 次 |
| `schtasks` / WMI `Win32_Process.Create` / `Start-Process` | ❌ 沙箱黑名单，明确提示不可绕过 | 2026-11-16 实测被拦 |

**为什么**：工具调用被包在 Windows 作业对象里，调用结束连带回收子进程；python 拉起 GUI 浏览器会触发额外限制直接 SIGTERM。而先由 shell 起 Edge 后，采集脚本走 `cdp_open.ensure()` 的**复用分支**（端口已通 → 直接 return，不 Popen），因此不会踩坑。

**端口 / profile / 登录态**：

| 项 | 值 |
|---|---|
| 端口 | 9223 |
| profile | `抓取脚本/edge-profile`（`cdp_open.PORT_PROFILE` 白名单：9222 jd-new-profile｜9223 edge-profile｜9225 tmall-profile） |
| 别名 | `<浏览器用户数据目录>` 与之**同一目录**（inode 相同，目录联接），用哪个路径都一样 |
| 登录态 | 持久在 profile 里（goofish 20 条 / bilibili 19 条 cookie），浏览器被回收**不用重新扫码** |
| 验证 | `curl -s --noproxy '*' http://127.0.0.1:9223/json/version` 返回含 `"Browser"` 即通 |

**分批**：单款约 98–130s（3 查询 × 2 页 + 切型号暂停 15–22s）。工具调用上限 10 分钟，**每批 3–4 款**最稳。

---

```bash
PY="python"
cd "<项目根>/skills/02_xy_price"

"$PY" run.py --layers --all                     # 全榜 E patch + 四层账（只读，写 patch 文件）
"$PY" run.py --layers --all --keys=SG Standard  # 单款
"$PY" run.py --layers --all --pool=../../快照/_skill2_pool_20261108.json   # 冻结池复现
"$PY" contract.py ../../原始数据/mb/_skill2_e_20261108.json --require-all
"$PY" contract.py --freeze ../../原始数据/mb/_skill2_e_20261108.json       # 改判据后重冻基线
"$PY" collect.py --mode=incr --plan-only        # 看要采哪些款（不联网，读全局锁）
"$PY" collect.py --mode=incr --run              # 真采 + 自动入池

# --- 视觉闸（0g）---
"$PY" vision.py --all --e-only --full            # 取图（断点续采，配额 300）
"$PY" ../../抓取脚本/_vision_make_shards_20261109.py --batch=b13   # 分片 55 条/片
"$PY" ../../抓取脚本/_vision_merge_shards_20261109.py              # 合并 v_bNN_MM.json → verdicts.json
"$PY" ../../抓取脚本/_vision_audit_20261109.py                     # 抓歪率审计
"$PY" ../../抓取脚本/_vision_gap_patch_20261110.py [--apply]       # 方案 A 回填（幂等）
```

---

## 9 双链路

| 环节 | `--mode=incr`（存量 116 款） | `--mode=new`（新琴） |
|---|---|---|
| 选款 | 映射到池的主榜款；自动挑：E 缺 / `n<门槛` / `span>60%` / 快照 >45 天 / `E_stale` | `--brand/--model/--query` 直给 |
| uid | 沿用 | 品牌须已登记，否则 **exit 2** |
| 采集 | 增量补数（`pages=2`、`detail≤10`） | 全链首采 |
| 落盘 | 自动入池（去重 `itemId` + 备份 + 守恒断言） | **入池 + 回写 uid 台账**（`sources.xy_pool`），**不写主榜**；另留 `_shadow_skill2_new_<ts>.json` |
| 排名 | patch 交 skill 07 | 不参与，观察区等人审 |
| 失败 | `reason_code` 下次重试 | 留档**不重试** |

---

## 10 当前状态（2026-11-11 05:15 终版封盘）

| 步 | 状态 | 证据 |
|---|---|---|
| P0 映射表 | ✅ | `_xy_keymap.json` **116/116**、`missing=0`；`negative` 24→23（`Squier\|CV Strat` 已进榜，不能再当负样本） |
| P1 契约 A1–A6 | ✅ | `contract.py` 对 `_skill2_e_20261111.json` **0 失败 0 警告**，**18 道闸**重冻（新增 `submodel`） |
| P2 账层 | ✅ | 116 款 / **96 有 E** / 24 缺口 |
| P3 采集入池 | ✅ 真机 | 11-11 轮入池：141 key 守恒、新增 5、覆盖 73、识图剔 21 |
| P4 new 影子 | ✅ 真机 | 11-08 `--mode=new --brand=PRS --model="SE DGT"` 全链走通 |
| P5 自检 | ✅ | `selftest.py` **116 款 / 有 E 96 / 0 漂移** |
| **P6 视觉闸 0g** | ✅ **全量** | 判读 **1,724** 条、拼板 1,718、待判 0；抓歪率 **5.8%**（93/1,617）；kill rate 视觉 17.4% + 型号错 3.0% |

**11-11 轮真正改动生产的三件事**
1. **Q1 封顶双源取小**：`cap = min(0.8×ref, 0.95×P)`，修掉「二手比一手贵」倒挂（榜3 6,400→6,069.6）；
   全榜封顶越界由 4 款降到 0 款。
2. **Q4 子型号词表 + 子品牌中文名拦截**：JS32R/TQ/Q/7/LH/DKAM、BJ 签名款、美标/60 周年、road worn 四类；
   另修 Gibson 池混入「易普峰 sg61」导致 E 从 7,840 掉到 3,420 的跨品牌污染。
3. **Q5/Q6 分层**：vision_gap 11 款按用户 11-10 方案 A 保持不动；无当期观测 5 款退观察区
   （主榜正式 95→**90**，MAX(O) 246.8187 不变）。

**历史遗留差异已清**（11-08 记录的 5 处）：X Soloist / LTD M-1000 / J Mascis / AV II '63 Tele / Am Pro II Strat
本轮全部由「池→主榜」同源重算覆盖，不再有「同源不同值」。

---

## 11 视觉闸（0g）落地规范

### 11.1 流水线

```
mb_pool_xy.json 候选
  → vision.py --all --e-only --full   取图（CDP 单进程串行，配额 300/轮）
  → 2×2 拼板 <itemId>.jpg + manifest.json
  → _vision_make_shards --batch=bNN   55 条/片
  → 4 路并行判读（每批 5 张 ViewImage，得出结论立即落盘 v_bNN_MM.json）
  → _vision_merge_shards              合并进 verdicts.json（读-改-写）
  → _pool_e_merge（drop.vision 生效）→ _board_e_resync → 主榜
```

### 11.2 判读口径（双轴，型号第一）

| 轴 | 取值 | 含义 |
|---|---|---|
| `model` | `ok` / `wrong` / `unsure` | 是否目标型号（含代际/代数） |
| `kind` | `personal` / `channel` / `not_photo` / `new` / `dup` / `swap` / `gen_wrong` | 卖家性质与图性质 |
| `verdict` | `ok` **仅当** `model==ok 且 kind==personal` | 其余写具体非 ok 值（`wrong_model`/`wrong_gen`/kind 值） |

`seen`（实际看到的品牌+型号）与 `why`（一句具体证据）必填；拼板混入推荐位/无关图必须显式写「混入非本商品图」。

### 11.3 取图三个坑（都会伪装成风控）

1. **JS 去重顺序**：先 `seen` 去重再判分辨率 → 220px 缩略图占位、原图被挤掉 → 全条 0 图。现为「先筛 `naturalWidth≥300`，再按 `base` 去重」。
2. **详情页等待**：5.4s 不够，图片区未渲染；现轮询至 12s。
3. **熔断误判**：连续 3 条取不到才 `[ABORT]`；此前应先单条探针（`_xy_risk_probe`）区分「死帖」与「风控/登录」。

### 11.4 方案 A：`vision_gap`（用户 2026-11-10 22:40 裁定）

视觉闸剔空、且旧 E 来自合法口径（channel 降权 cap 0.50 / 弱样本 / 参考级）的款：

- 池：`E_final` 沿**主榜现值**，`E_final_grade="vision_gap"`、`E_vision_gap=true`、区间置空（不与旧点估计同源）；
- 主榜：**E 与数值一律不动**，名次只随其他款自然浮动；
- 实现：`抓取脚本/_vision_gap_patch_20261110.py`（幂等，可重跑）；当前 14 款（13 款在主榜）。
- 处置出口：进下一轮定向补采；补到 ≥3 条个人样本后自动转回正式级，`vision_gap` 标记清除。

### 11.5 主榜写入唯一路径

`_board_e_resync_YYYYMMDD.py`（**整榜一次性，禁逐款**）：E 只取正式级 → U/V/O 用生产同一公式 → 五道守卫（条数 116 / V链复现 / O 复现 / MAX(O) 变动需 `--accept-base-shift` / rank 连续）→ 才写盘。

**11-11 起追加两道**：
- **退观察区**：无当期观测（参考级 / 无样本 / 历史沿用）且非 vision_gap 的款 → `boardStatus=观察`、rank 置空、E 保留为历史值（DS Q6=A）；
- **背离守卫**：`n_personal≥3` 时若 `|E − 个人中位| / 个人中位 > 40%` → 本期**不写**，保留旧值并进待查清单（实证 Gibson SG Standard：个人中位 7,549 / E 被渠道拉到 3,420）。

### 11.6 E 封顶（Q1，2026-11-11）

```
cap = min(0.8 × ref, CAP_COEF_P × P)      # CAP_COEF_P = 0.95，配置在 _pool_e_merge
```
- `ref` = 人工配置的行货/官方价；`P` = 一手实付（京东券后）。**两者语义不同，不可互相替代**。
- 病灶：ref 比 P 高 25% 时，`0.8×ref` 会**高过 P** → 榜单出现「二手比一手贵」（榜3 实证 6,400 vs 6,389）。
- 只改一处：`merge_one` 把 `p_proxy` 传入 `min(ref, 0.95P/0.8)`，`e_channel.adjust` 判据文件不动。
- 留痕字段：`cap_p_value`（0.95×P）、`cap_p_applied`（本轮是否被 P 侧压下来）。
- 回滚：`CAP_COEF_P = None` 即回到旧口径 `0.8×ref`。

### 11.7 子型号 / 子品牌词表（Q4，2026-11-11）

| 规则 | 拦什么 | 实证 |
|---|---|---|
| `SUBMODEL_DENY` | Jackson JS32 的 `R/TQ/Q/7/LH/DKAM/DK3XR`；Epiphone LP Junior 的 `Billie Joe Armstrong` 签名款；Fender `American Standard/美标/60周年`；`road worn/重度做旧` | 这些样本视觉判 ok（图确实是同品牌同琴型），但**不是同一款** |
| `_UPGRADE_CTX` 守卫 | 标题含「升级/改装/更换/拾音器」时**放行**「美标」 | 实证「墨标 st 升级美标拾音器 3,887」是真目标款 |
| `child_brand_cn` | 搜母品牌时出现子品牌**中文名**（易普锋/易普峰/依霹风/斯奎尔…） | Gibson 池混入 3 条「易普峰 sg61…SG Standard」→ E 7,840→3,420 |

计数键独立（`submodel` / `brand_mismatch`），便于契约 A6 单独看 kill rate；词表是显式表，**禁模糊匹配**。

---

## 12 用户已定案（2026-11-08 12:5x，追认 DS 建议；2026-11-10 追加 A9）

| # | 事项 | 定案 |
|---|---|---|
| A6 | 闸门回归阈值 | **按冻结基线相对判**：某闸 kill rate 上升 **>5pp 阻断**；绝对 >20% 仅告警（`hard_new` 天然 27.6%，绝对阈会每轮判红） |
| Q1 | 重采语义 | 当期只用**最新快照**；旧值存 `snapshots[]`，**只留最近 3 期** |
| Q2 | 快照过期 | **45 天**，按「距上次全量采集日」计 |
| Q3 | E patch 落主榜 | **只走 skill 7**；`_recalc_e_batch --write` 永久禁用 |
| Q4 | n=2 两点中点 | 保留，标 `D_不排名` |
| Q5 | 池内 24 个未入榜 key | 保留不重采，记 `out_of_board` |
| Q6 | E 与 P 时点差 | 同自然月可接受，跨月重采 |
| Q7 | `_diag_kept` 的 `_xy_re` 覆盖 | 删除，改注册表驱动（排在 P3 自动入池之前） |
| **A9** | 视觉闸剔空后的旧 E（2026-11-10） | **方案 A**：沿用旧值 + 标 `vision_gap`，主榜 E/数值不动；补采达标后自动转回正式级。**不**置空退出排名、**不**立即整款补采 |
| **A10** | E 封顶基数（2026-11-11，DS 代拍板 Q1） | `min(0.8×ref, 0.95×P)`，一处生效（`merge_one`），配置化可回滚。**不**改 ref 语义、**不**加出口守卫 |
| **A11** | 子型号混入（2026-11-11，Q4） | 显式词表 + 上下文守卫，只剔确定性错款；视觉闸「同品牌同琴型即 ok」的粒度问题记为下轮改造 |
| **A12** | 无当期观测款（2026-11-11，Q5/Q6） | vision_gap 11 款**保持方案 A 不动**；参考级/无样本/历史沿用 5 款**退观察区**（E 保留历史值、不排名）；n=2/3 弱样本保留并标低置信 |
