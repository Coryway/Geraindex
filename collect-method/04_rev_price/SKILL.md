# skill 04 · Reverb 海外二手价（S）— v1.1（2026-11-13 封板：补 run.py 统一入口）

> 按 `_过目/归档/148_skill封装规范` 七段封装。
> **口径由 DS 在 2026-11-12 裁定**（见 `_过目/215`，6 条改判 + 7 个补漏，全部落在本文 §9）。
> **本 skill 不写主榜**。唯一写入口是 skill 07。采集只出池 + provenance，落榜另开一轮。

```
skills/04_rev_price/
├── SKILL.md        七段契约（本文件）
├── config.json     汇率 / 代理候选 / 节流 / 采纳门槛 / 闸门开关
├── registry.py     116 款查询词与型号闸（唯一真源，生成 _rev_registry_*.json）
├── gates.py        分类闸（纯函数，可单测）
├── collect.py      采集内核（--probe / --only / --limit / --pages）
├── contract.py     A1–A6 断言（落盘前跑，不过 exit 2）
├── selftest.py     闸门回归（35 项，把 2026-11-12 修过的每个 bug 固化成断言）
└── reconcile.py    影子对照（池 vs 主榜，只读，不改榜）
```

---

## 1 输入契约

| 输入 | 路径 | 关键字段 | 缺失怎么办 |
|---|---|---|---|
| 注册表 | `原始数据/mb/_rev_registry_<YYYYMMDD>.json` | `items[uid].{key,query,hit,excl,tier}` | 缺 → `python registry.py` 重建；单款缺 → `[skip]` 并计数 |
| 采集配置 | `skills/04_rev_price/config.json` | `fx/api/proxy/pacing/adopt/gates` | 缺 **exit 2**（不许用默认值兜） |
| 主榜（只读） | `MultiBrand_Ranking_Data.json` | `key/rank/value.u_breakdown.{S,P_os,wS}` | 缺 → 只出池，`reconcile.py` 不跑 |
| 历史池（仅对账） | `mb_pool_rev.json` / `reports/_rev_fill37_20261106.json` | `median_cny` | 缺 → 跳过旧值对比，不影响采集 |
| UID 映射 | `原始数据/mb/mb_uid.json` | `mainboard_key → uid` | 缺 → `registry.py` 用 key 哈希兜底并告警 |

**映射铁律**：`uid` 是唯一主键，`key` 只用于展示与对账。禁止用 key 模糊匹配回填。

**时点契约**：每条记录必带 `capturedAt`（本地时间，分钟精度）+ `src`（产物文件名）。
S 是在售挂价的**时点快照**，隔日必然漂移，禁止跨日混算、禁止拿新 S 覆盖旧 S 而不留痕。

---

## 2 输出契约

### 2.1 采集池 `原始数据/mb/_rev_price_<YYYYMMDD>.json`

```jsonc
{
  "FEN-001": {
    "uid": "FEN-001", "key": "MIJ Hybrid II Telecaster",
    "query": "Fender MIJ Hybrid II Telecaster", "hit": ["hybrid ii tele"], "tier": "C",
    "src": "_rev_price_20261112.json",            // 必须等于本文件名（A4）
    "proxy": "http://127.0.0.1:60108",           // 采集时实际用的代理，排障用
    "capturedAt": "2026-11-12 18:31",
    "total": 70, "fetched": 70, "pages": 1, "coverage_pct": 100.0,
    "bucket_count": {"s": 26, "p_os": 9, "drop": 35},
    "drop": {"limited": 30, "dup": 2, "model_miss": 1, "lefty": 1, "parts": 1},
    "stats": { /* pool_stats 全字段，见 §4.3 */ },
    "outlier": {"idx": [], "method": "log_mad_3.5", "mad": 0.04, "thr": 0.21},
    "fx": 6.69, "fx_date": "2026-11-10",
    "s_usd": 1359.0, "s_cny": 9095,
    "p_os_usd": 1448.0, "p_os_cny": 9687,
    "s_over_p_os": 0.939,
    "n": 26, "adopt": "adopted", "s_basis": "adopted",
    "p_os_n": 9,
    // probe 三态：state / http / total / zero_in_sample
    // provenance: "live"=采集时现场写（唯一可信）；"inferred"=事后从 raw_status 反推
    "probe": {"state": "capped", "http": 200, "total": 70, "zero_in_sample": false,
              "provenance": "live"},
    "items": [ /* 逐条保留：id/title/cond/shop/seller/mint/price_usd/created */ ]
  }
}
```

> **`provenance` 是硬要求（DS 复验条件 1）**：只有 `live` 才算现场采集。
> `inferred` 的 probe 字段**可复算但不可宣称验收闭环**，前端不得据此显示「验收通过」类标识。
> 历史海外值（2026-10 前抓的）一律另存 `s_legacy_cny`，**不得**占用 `s_basis`、不得进 patch（A12）。

**幂等规则**：同一 `DATE` 重跑 = **覆盖**该 uid 记录（`--only` 也是合并写入，不是重建池）。
跨日 = 新文件。**禁止**把不同 DATE 的池合并成一份。

### 2.2 溯源 `原始数据/mb/_rev_provenance_<YYYYMMDD>.json`

每款 12 字段快照：`key/query/src/capturedAt/total/fetched/n/s_usd/s_cny/adopt/tier/coverage_pct`。
用于「S 从哪来」的审计，不参与计算。

### 2.3 原始层 `原始数据/mb/raw_rev/<uid>_<YYYYMMDD>.json`

```
{"uid","query","pages":[{"_page":1,"_status":200,"_listings":[...原文...]}]}
```
**保留 API 原始返回**，用于离线重放调闸（改 `gates.py` 不必重抓）。
`_status != 200` 的页也写进去（`_listings` 缺失），保留失败证据。

### 2.4 skill 07 的输入形状

`reconcile.py --emit` 产出 `_rev_s_patch_<YYYYMMDD>.json`：
```jsonc
{"FEN-001": {
  "key": "MIJ Hybrid II Telecaster",
  "s_cny": 9159, "n": 25, "adopt": "adopted", "s_basis": "adopted",
  "p_os_cny": 9689, "p_os_n": 9, "p_os_usd": 1448.27,
  "p_os_note": "brand_new 挂单中位，单点值不参与置信度标注",
  "retention_capped": true,      // cap 咬住 → 名次与 S 解耦，必须随 patch 进主榜
  "robust_ok": false,            // false = 不参与任何稳健性结论
  "robust_note": "capped：名次由 P 决定，与 S 解耦",
  "addr": {"src": "...", "capturedAt": "...", "probe": "capped",
           "coverage_pct": 100.0, "total": 70, "fx": 6.69, "fx_date": "2026-11-10"},
  "flags": ["shop_heavy"]
}}
```
**只含 adopt ∈ {adopted, low_n} 的款**。`thin` / `none` / `probe_failed` 不进 patch，但要在报告里列出来。

`robust_ok` 的判定（DS 验收条件 7）：`adopt=adopted` 且 `n≥10` 且 `retention_capped=false`。
不满足的一律不参与「前 N 名稳健」这类断言——包括 `low_n` 与 8 条样本刚过线的款。

> **`n≥10` 这个数是工程约定，不是统计推导出来的。** 出处与理由：本轮在榜 95 款的 n 中位 14，
> n 5–9 共 **14 款**、n<10 共 **41 款**；前 30 名相邻 ΔT 中位仅 **0.22 分**，
> n<10 时中位数的抽样波动足以吃掉整个 T 差值，所以把它们排除在稳健性结论之外。
> 但**没有做 bootstrap 标定**——下一轮用 n 分档 bootstrap 把这条阈值换成实证值，届时改这里。

**下游三条硬约束**（DS 2026-11-12 复验要求，skill07 与前端都必须遵守）：

1. 字段互斥：`s_basis` / `s_cny` / `s_legacy_cny` 在合并层入口先断言，违反 exit 2（A12）。
2. robust 隔离：`retention_capped=true` 或 `robust_ok=false` 的款**禁入主榜排序与任何「稳健」结论**，
   只进附注栏并带 `robust_note`；`sens_excluded=true` 的款另记「S 变了但不传导」。
3. provenance：`probe.provenance=inferred` 的字段，前端不得显示「验收通过」类标识；
   敏感性数字同屏必须带「情景假设，非误差界」。

---

## 3 风控参数（不可放宽）

| 项 | 值 | 原因 |
|---|---|---|
| 并发 | **1（单进程串行）** | 并发会触发 Reverb 429，重试成本高于串行 |
| 页间隔 | 2.5–4.5s 随机 | 实测 <2s 连续请求 3 次内必 429 |
| 款间隔 | 1.5–3.0s 随机 | 同上 |
| 重试 | 3 次，退避 6/12/18s | 429 恢复窗口约 10s |
| 超时 | 30s | 代理抖动时 15s 会误判失败 |
| 代理 | **必须走 VPN**，直连 403 | 端口每轮变，按 `config.proxy.candidates` 顺序探活 |
| 403 语义 | **= probe_failed，绝不等于 0 条** | 把 403 当 0 条会静默把 S 写成 none |

代理探活失败 → `SystemExit`，**不许降级直连**。

---

## 4 三道闸（L1 采集时）

> 归属：**全部在 L1（采集时）**。L2（skill 07 合并）不再重复判，只读 `adopt`。

### 4.1 结构闸（先判，命中即 drop）

| # | 闸 | 判据 | 理由码 |
|---|---|---|---|
| 1 | not_live | `state.slug ∉ {live, draft}` | `not_live` |
| 2 | not_guitar | `categories[0].full_name` 不以 `Electric Guitars` 开头 | `not_guitar` |

### 4.2 型号闸（顺序敏感）

| # | 闸 | 判据 | 理由码 |
|---|---|---|---|
| 3 | model_miss | `hit` 各项拆 token，**全部 token 出现**（≥3 字符允许前缀，≤2 字符要求整词） | `model_miss` |
| 4 | other_model | 命中款级 `excl` 任一子串 | `other_model` |
| 5 | other_model | 命中 `GLOBAL_EXCL`：`reissue / inspired by / igc / maestro / vibrola / 19[5-6]x / custom shop` | `other_model` |

> **GLOBAL_EXCL 的合法性**：2026-11-12 实测，116 款 key 中命中这组词 **0 款**，即对全榜一律意味着「另一个更贵的产品」。

### 4.3 文案闸（两级，顺序在型号闸之后）

| # | 闸 | 判据 | 理由码 |
|---|---|---|---|
| 6 | nonguitar_type | 命中 `NONGT`（bass / acoustic / amp / pedal …） | `nonguitar_type` |
| 7 | parts（硬） | 剥掉捆绑短语后命中 `HARD_PARTS`：`pickguard / neck only / body only / for parts / parts guitar / parting out / project guitar / relic kit / template` | `parts` |
| 8 | parts（软） | 剥掉捆绑短语后命中 `SOFT_PARTS` **且** 标题不含 `electric guitar` | `parts` |
| 9 | lefty | `left[ -]?hand / lefty` | `lefty` |
| 10 | limited | `limited / fsr / special run / exclusive / signature / anniversary / dlx / deluxe` | `limited` |
| 11 | b_stock | 标题命中 `BSTOCK` **或** condition 含 `b-stock / non-function` | `b_stock` |
| 12 | modded | `custom shop / modified / upgraded / refinished / partscaster / kit` | `modded` |
| 13 | auction | `auction == true` | `auction` |
| 14 | no_price | `price.amount` 取不到 | `no_price` |

### 4.4 分桶（闸全过之后）

| 桶 | 条件 | 去向 |
|---|---|---|
| `p_os` | `condition.slug` 以 `brand-new` 开头 | 海外一手价 P_os 桶，**不进 S** |
| `s` | 其余（含 Mint） | S 池，`meta.mint=true` 标记 |

> **order 为什么必须是「型号/品类在前，brand_new 在后」**：Reverb 上「全新 Telecaster」会搜出大量别的型号新货，
> 若先按 brand_new 分桶，会把别的型号的新价塞进本款 P_os。

### 4.5 去重

键 = `(shop 归一化, price)`；`shop` 为空时退回 `("", price, norm(title)[:40])`。

> **为什么加 shop**：同一商家挂 5 个配色、同价，不构成 5 个独立市场信号。
> 实测 YAM-008 有 5 条 `$929.99` 同商家不同色，把中位从 ~$790 抬到 $813（+2.9%）。

### 4.6 统计（**只报警，不自动剔**，DS Q3）

`log-MAD`：`|log p − median| > 3.5 × 1.4826 × MAD` → 记 `outlier.idx`，**不动中位**。
`n < 5` 不判（`method=skipped_lt5`）。`MAD=0` 退化为 `max/median > 2.5`。

诊断 flags（写进 `stats.flags`，供人工复核，**不改数值**）：
`shop_heavy`（商家占比 > 0.7）/ `mixed`（商家与个人中位差 > 30% 且两组各 ≥3）/ `mint_premium`（Mint 溢 > 20% 且 ≥3 条）。

---

## 5 自校验 A1–A6（落盘前跑，不过 exit 2）

| # | 断言 | 说明 |
|---|---|---|
| A1 | `fetched == s + p_os + Σdrop` | 条数守恒，含 dup |
| A2 | `min ≤ median ≤ max` 且 `n == len(items)` | 中位自洽 |
| A3 | `max/median > 3` 必须已在 `flags` 或 `outlier.idx` 里 | 长尾必须留痕 |
| A4 | `query / hit / src` 非空，且 `src` 等于本文件名 | 溯源不可缺 |
| A5 | `n == 0` 时 `s_cny` 必须为 null，`adopt ∈ {none, probe_failed}` | 禁止写 0 元 |
| A6 | 116 款全覆盖，每款落在 `adopted / low_n / thin / none / probe_failed` 五态之一 | 不许漏款 |
| A7 | `probe.state ∈ {failed, no_listing, capped}` 必填 | 三态不可省，否则「没连上」会被当「没货」 |
| A8 | `state=failed` → `adopt=probe_failed` 且不带 S | **403 ≠ 0 条** |
| A9 | `adopt=none` 只允许两种来路：`no_listing`（total=0）或 `capped` 且 `zero_in_sample` | 分清「真空集」与「样本内筛空」 |
| A10 | `s_basis` 与 `adopt` 一致，`none` 可细分为 `none_zero_in_sample` | 五态可追溯 |
| A11 | `s_basis=none` 只许由 `probe.state=no_listing` 派生 | 脚本硬断言，不靠约定；`none_zero_in_sample` 语义是「**供给存在但样本不足**」，下游不得当无供给 |
| A12 | patch / 加权 / 排序 / 稳健性输出里**不许出现** `s_legacy_cny`；`s_basis∈{none,none_zero_in_sample}` 不许带 `s_cny`；`probe` 必带 `provenance` | `contract.check_patch()`，违反 exit 2，不留警告 |

> **本轮 provenance 声明（DS 验收条件 3）**：`probe` 字段是 2026-11-12 由 `_rev_backfill_probe_20261112.py`
> 从 `raw_rev/*.json` 的 `_status` 与池内 `total` **事后反推**的，不是采集时现场写的
> （因为池是在三态字段落地之前抓的）。字段可复算 ≠ 现场采集，**本轮不得宣称 probe 验收闭环**；
> 下一轮采集必须现场写 `probe`，并保留原始 `_status/total`。

**probe 三态（2026-11-12 回填，实测分布）**：

| state | 判据 | 本轮 116 款 |
|---|---|---|
| `failed` | 首包 status ≠ 200 | 0（全量采集零请求失败） |
| `no_listing` | total = 0 | 14 |
| `capped` | total > 0，只看了前 N 条 | 102（其中 8 款样本内筛空 → `none_zero_in_sample`） |

**采纳门槛（DS Q2 三态）**：

| n | adopt | 是否进 patch |
|---|---|---|
| ≥ 5 | `adopted` | 是 |
| 3–4 | `low_n`（强制标记） | 是，但必须带 `low_n` 标记 |
| 1–2 | `thin` | 否，只进池 |
| 0 | `none` | 否 |
| 请求失败 | `probe_failed` | 否，且**不许写 none** |

---

## 6 已知坑（全部有实证，2026-11-12 实测）

| # | 坑 | 实证 | 现状 |
|---|---|---|---|
| E1 | `parts` 闸把「with Gig Bag」的整琴全剔 | YAM-008 剔 26/100、EPI-001 剔 17/100，S 池直接归零 | 已修：`BUNDLE` 剥离 + 两级配件闸 |
| E2 | `w/case`、`w/bag`（无空格）没被剥离 | 修完 E1 后仍有 6 条漏网 | 已修：`w/`、`w.` 单独分支，不加后置词边界 |
| E3 | 「2 P90 Style Pickups」是整琴配置，被当卖拾音器 | YAM-008 误杀 2 条 | 已修：软闸加了 `WHOLE_HINT`（标题含 `electric guitar` 则不判软闸） |
| E4 | `hit` 用连续子串，`hybrid ii tele` 匹配不到 `Telecaster Hybrid II` | FEN-001 误杀 10/70 | 已修：`hit_ok()` 词袋 AND |
| E5 | `b_stock` 只判标题，漏掉 condition 已是 B-Stock 的单 | YAM-008 4 条 `$836.99` 混进 S 池 | 已修：加 condition slug 判定 |
| E6 | 同商家多配色同价挂单被当独立样本 | YAM-008 5 条 `$929.99` | 已修：去重键加 shop |
| E7 | 混入「1964 SG Standard Reissue / Inspired by Gibson」 | EPI-001 3 条 `$999–1299`，中位从 ~$490 抬到 $507 | 已修：`GLOBAL_EXCL` |
| E8 | 正主（Casino）中位 $1,100 明显偏高，因混了别的型号 | 11-06 历史事故 | 靠 `GLOBAL_EXCL` + `excl` + 人工审计三层防 |
| E9 | `--only` 落盘直接覆盖整个池 | E21 踩过两次 | **已避免**：`pool = 读旧池 → pool[uid] = rec` 合并写 |
| E10 | 国产四家在 Reverb 常年 0–2 条 | Shijie / Keipro / Farida / Tagima | 探一次确认即可，不必反复采 |
| E11 | `hit` 比型号宽，把另一个更贵的产品线吃进 S 池 | `standard stratocaster` 命中 `American Standard Stratocaster`，FEN-005 池内 23/79 条是美标，中位从 $610 抬到 $725（+15.9%） | 已修：这两款 `excl` 加 `american / usa`；A3 长尾警告会先亮，别忽略 |
| E12 | 自建请求头 UA 只写 `Mozilla/5.0` → 边缘 WAF 返 403 | 403 的响应体是 **HTML 首页**不是 JSON 错误，极易被误判成「限流」，白等半小时 | 一律复用 `collect.Net`（UA 写全 `Mozilla/5.0 (Windows NT 10.0; Win64; x64)`）；判 403 先看 body 首字节是不是 `{` |
| E13 | API 默认排序**就是 relevance**（已验证 2026-11-12） | 默认序与 `sort=relevance` id 重合 20/20（仅次序微抖）；`price\|asc` / `published_at\|desc` 重合 0/20 | 不要加 `sort` 参数；「前 150 = 最相关」这个前提成立 |
| E14 | 短时并发请求会触发临时封禁 | 连打约 10 个请求后全部 403，等待约 10 分钟后自行恢复 | 严格串行 + 页间隔 2.5–4.5s；不要用脚本做参数扫描 |
| E15 | **同一款琴的闸门在主榜侧修过、S 侧没同步** | `Epiphone\|Firebird`：主榜 note 记着 2026-11-11 已剔 Firebird I/III/VII/IGC，但 registry 的 `excl` 只有 `studio/special`，S 池混入 8 条 $1,000–1,485 的其它型号，中位被抬高 4.9% | 改任何一款的型号闸门时，**先 grep 主榜 `note` 看有没有历史修复记录**；`Firebird V` 是标准款不能剔 |
| E16 | `registry.excl_re_of()` 会覆盖 TABLE 第 4 段 | 在 TABLE 里写了正则却完全不生效，排查半小时；根因是 `excl_re_of(key)` 无条件返回年份默认值 | 已修为「TABLE 显式正则 > `_EXCL_RE_OVERRIDE` > 年份默认」；改完必须回读 `_rev_registry_*.json` 确认写进去的是新值 |
| E17 | `S/P_os > 0.95` 护栏在 P_os 统一后会真触发 | 统一成海外全新价后 8 款超线、4 款 >1.00（最高 Shijie STE 1.49，海外二手比国内新价贵 49%） | 回退 `S=E` 时 **U 直接取 E×W，不许再乘 `P/P_os`**——E 已是国内口径，再折算是双重折算 |

---

## 7 失败处理（禁止 `continue`）

| 情形 | 写什么 | 不许写什么 |
|---|---|---|
| 代理探活全失败 | `SystemExit("[net] 无可用代理")`，**整轮不跑** | 不许降级直连、不许跳过 |
| 单款首页 403/超时 | `adopt=probe_failed` + `probe_status`，`n=0`，`s_cny=null` | 不许写 `none`（那是「真 0」） |
| 单款第 2 页起失败 | 保留已取到的页；`coverage_pct` 如实反映；`n` 按实得算 | 不许把「取到一半」当全量 |
| 型号闸全灭（`s=0` 且 `drop.model_miss` 很大） | `adopt=none` + 高 `model_miss` 计数 | 不许判定「该琴无二手」——要先查 query/hit 是否写错 |
| 断言 A1–A6 不过 | `exit 2`，池**仍落盘**（便于排障） | 不许静默通过 |

---

## 8 命令

统一入口 `run.py`（只调度，不重写逻辑）；底层脚本也可单独调。

```bash
cd <项目根>

# 0 只读全链路：闸门回归 → 契约 → 影子对照（不改榜，日常验收用这个）
python skills/04_rev_price/run.py --all

# 1 闸门回归（45 项，改过 gates.py 必跑）
python skills/04_rev_price/run.py --selftest

# 2 契约 A1–A12 全量校验（116 款）
python skills/04_rev_price/run.py --contract

# 3 影子对照 + 出 skill7 patch
python skills/04_rev_price/run.py --reconcile --emit

# 4 扰动：S 中位 ±20%，看名次变动 >5 的款
python skills/04_rev_price/run.py --reconcile --perturb

# 5 主榜 S 与产物一致性审计
python skills/04_rev_price/run.py --audit

# 6 重建查询注册表（改了 registry.py 之后必跑）
python skills/04_rev_price/run.py --registry

# 7 探活（先跑这个，确认 VPN 与端口）
python skills/04_rev_price/run.py --probe

# 8 单款排障（合并写入，不会清空池）
python skills/04_rev_price/run.py --only FEN-001 --pages 2

# 9 试跑 10 款
python skills/04_rev_price/run.py --limit 10

# 10 全量 116 款（约 25–40 分钟，单进程，须开 VPN）
python skills/04_rev_price/run.py
```

`--all` 的验收输出应是：`自检 45 项全部通过` + `契约通过（116 款 0 错）` + `对齐 95/95`。若主榜 S 与本产物已同步，影子会显示「名次变动 0 款」，这是正常状态（说明产品与榜一致），不是没跑。

---

## 9 口径（DS 2026-11-12 裁定，改动须经用户）

### 9.1 六条主口径

| # | 口径 | 落地 |
|---|---|---|
| C1 | **S = 在售二手挂价中位 × 6.69**，含 Mint，不含运费税 | `gates.classify` + `stats.median_usd` |
| C2 | **主 S 不拆商家/个人**，但必采拆分明细 | `stats.{shop_median, private_median, n_shop, n_private, shop_share}` |
| C3 | **汇率固定 6.69**（`fx_date=2026-11-10`），禁止实时汇率重算历史 | `config.fx` |
| C4 | **n≥5 主替换 / 3–4 标 low_n / 1–2 只进池** | `contract.decide_adopt` |
| C5 | **不直接改榜**，先出影子 | `reconcile.py` |
| C6 | **异常值只报警不自动剔** | `gates.flag_outliers` |

### 9.2 DS 的 6 条改判（覆盖初版计划）

| 问题 | 初版 | 改判 |
|---|---|---|
| Q1 卖方结构 | 「两边都是挂价所以可比」 | 论证不成立（只解决价格类型，没解决卖方结构/议价/流动性）。**主 S 不拆**避免 private 子样本打爆方差，但必采拆分字段；`shop_share>0.7` 或中位差>30% 且两组 ≥3 → 标 `mixed` |
| Q2 样本门槛 | n≥3 否则留旧值 | **三态且不许留不可追溯旧值**。「留旧值」= 把不可审计误差藏进榜单 |
| Q3 异常值 | `max/median>3` | **只报警不自动剔**；业务闸优先；剩余用 log-MAD；n<5 只打 flag |
| Q4 mint | 全剔 | **纳入**。全剔 → S 偏低 → U 偏低 → R 低 → V 高 → **O 低（低估性价比）** |
| Q5 探 0 | 0 条写 none | 照做，但**必须区分真 0 与失败**（403 ≠ 0 条）；`wS=0` 不是无损，跨款可比性会被破坏 |
| Q6 P0 追溯 | 80 款逐款指认 | **缩到 2h**：冻结 baseline + 32 款可验证的解释模板，**48 款直接标 `unverified`**，不给假数据做考古 |

### 9.3 七个补漏（已落地项标注）

| # | 环节 | 状态 |
|---|---|---|
| 1 | 汇率带 `fx_date/fx_source`，禁止今日汇率重算历史 | ✅ `config.fx*` |
| 2 | raw 层落盘（原始 JSON / 分页 / 代理 / HTTP 状态 / 时间戳） | ✅ `raw_rev/` |
| 3 | 记录 `total` 与 `coverage_pct`（只抓首页会引入选择偏差） | ✅ `rec.total / coverage_pct` |
| 4 | 时点性：榜单须带采集日期 + 版本 | ✅ `capturedAt` |
| 5 | 扰动测试：S 中位 ±20%，看名次变动 >5 的款 | ⏳ `reconcile.py --perturb` |
| 6 | 监控：成功率 / 0 结果率 / 中位漂移 / doubt 率 / shop_share | ⏳ 报告层输出 |
| 7 | 限流与失败态：403 ≠ 0 条 | ✅ `probe_failed` |

---

## 10 变更表

| 日期 | 版本 | 改了什么 | 谁的指令 |
|---|---|---|---|
| 2026-11-12 | 1.0 | 首版封装。闸门重构（两级配件闸 + 全局异型 + 词袋型号匹配 + b_stock 按 condition）；`max_pages` 6→3（**Reverb API 硬顶 per_page=50**，传 100 也返回 50，故 3 页 = 150 条上限，请求数减半而覆盖上限不变） | 用户「开始 skill4 封装，计划先给 DS 审」 |
