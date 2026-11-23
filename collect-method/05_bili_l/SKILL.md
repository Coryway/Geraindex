# skill 05 · B站评论旁证（L）— v1.0（2026-11-14 封装）

B站评论 → T/W/P/N 四分类 → `L_ai`。判定逻辑在 `skills/_judge_core/`，与 skill6 共用同一份定义、
同一个 parse、同一个 L 公式。**本 skill 只负责采集、契约、对账。**

---

## 1 输入契约

| 输入 | 位置 | 说明 |
|---|---|---|
| 采集产物 | `原始数据/mb/_bili_c_final_20261114.json`（config 可换） | `{key: {query, videos:[{bvid,title,replies:[{rpid,like,msg}]}], capturedAt}}` |
| 注册表 | `skills/05_bili_l/registry.json` | 116 款查询词 + 目标词（型号守卫用） |
| 参数 | `skills/05_bili_l/config.json` | 采集节奏、闸门阈值、judge 版本 |

输入为空或登录态失效 → **停**，不续采、不重试。

## 2 输出契约

`原始数据/mb/_bili_l_<YYYYMMDD>.json`

```json
{"_meta": {"platform":"bili","judgeVer":"2.0.0","promptSha":"29d8bd91…",
           "variant":"v0","gate_enabled":true,"source":"…","sourceSha256":"…",
           "known_missing":["…"],"states":{"adopted":88,"thin_l":9,"no_video":18}},
 "records": {"<key>": {"n":46,"n_raw":46,"miss":0,"counts":{"T":3,"W":5,"P":12,"N":26},
                        "rate_T":6.52,"L_ai":69.64,"thin_l":false,
                        "judgeVer":"2.0.0","prompt_sha256":"29d8bd91…",
                        "model":"deepseek-chat","cacheHit":true,
                        "state":"adopted","samples_T":[…],"gate":{…}}}}
```

`L_ai = min(100, 50 + 100×(P−T)/(n+10))`，K=10。

`n_raw` = **去重后**待判条数；`n_input` = 去重前；`n_dedup` = 去重去掉的条数；`miss` = 判定失败条数。
契约要求 `n_raw − n == miss`（去重不算 miss）。

## 3 风控参数（不可放宽）

| 参数 | 值 | 原因 |
|---|---|---|
| 视频/款 | 1–4（历史 1） | B站一款常常只有 1 个测评；多抓会引入翻弹与广告 |
| 页/视频 | 3（×20） | 60 条是历史封顶，P1–P3 冻结 |
| 页面间隔 | 8–12s | 过快会掉登录态 |
| 款间隔 | 20–30s | 同上 |
| 日配额 | ≤200 视频 | 过线必被封 |

## 4 三道闸

- **L1 采集**：CDP 9223 + `edge-profile` 的 SESSDATA。匿名 API 返 412（需 wbi 签名），
  所以必须在页面上下文里 fetch。踢到 412/未登录 → 停。
- **L2 视频类型闸**（D2）：剔 翻弹/开箱/二手/改装/教学/谱；收 测评/对比/复刻/长期；未知保留标 `needs_review`。
- **L3 契约**：A1–A12，见 `contract.py`。

## 5 自校验

```bash
python skills/05_bili_l/run.py --selftest     # 20 款金标准，counts 与 L 必须逐项一致
python skills/05_bili_l/run.py --contract     # A1–A12
python skills/05_bili_l/run.py --all          # 注册表 → 自检 → 契约
```

金标准基线：`_selftest_baseline.json`（`--freeze` 生成）。模型升级后第一个在这里报警。

## 6 已知坑

| 坑 | 现象 | 处置 |
|---|---|---|
| 匿名 API 412 | 返回 HTML/空 | 必须走页面上下文；`--probe` 先验 |
| SESSDATA 过期 | 全挂 | 停，重新登录，不重试 |
| 灌水评论 | 「催更/抽奖/BGM是啥」被误判 P | prompt 第 5 条已写死判 N |
| 1 视频/款 | 闸门误杀 → 该款 L 归零 | B站闸门**全量人工复核**，不自动剔 |
| 评论折叠 | 负面被折叠 → L 偏高 | `audit` 记录 `is_folded` |
| 排序依赖 | 打乱输入顺序 → 分批变化 → 标签变化 | 规范序 = **冻结源清单的物理顺序**（源文件序 → 视频序 → 评论序）；清单指纹由 A11 阻断 |
| **API 非确定** | 同一份输入连跑两次，个别款差 1–2 条评论（temp=0 也如此） | 实测噪声底：12 款 × 2 次，e p50=0 / p95=0 / max=1.0，ΔL=0 占 92%。**只能靠缓存冻结**，不承诺重算逐字节相同 |

## 7 失败处理（禁止 `continue`）

| 情形 | 动作 |
|---|---|
| API 失败 | 单批重试 3 次；仍失败 → `fail+=1`，不写缓存，记录 `judge_failed` |
| 解析失败 | 同上，`miss` 必须写明原因（A2） |
| 闸门剔空 | `state=no_video`，`fail_reason=gate_dropped_all`，不入对账 |
| 登录失效 | 整批停，不进下一款 |

## 8 命令

```bash
cd <项目根>

python skills/05_bili_l/run.py --registry          # 重建注册表（改了 registry.py 之后）
python skills/05_bili_l/run.py --probe             # 验 CDP 端口
python skills/05_bili_l/run.py --collect --keys "LTD|EC-1000"
python skills/05_bili_l/run.py --judge             # 判定（有缓存则秒回）
python skills/05_bili_l/run.py --judge --no-gate   # 等价性验证专用：关闸 = 历史评论集
python skills/05_bili_l/run.py --reconcile         # 出 L patch 给 skill7
python skills/05_bili_l/run.py --all
```

## 9 与 skill4 的差异（不能照抄）

1. skill4 是 HTTP 直连，本 skill 依赖浏览器登录态 —— 失败模式不同：skill4 可重试，本 skill 只能停。
2. skill4 产物是确定的价格，本 skill 产物是 LLM 判断 —— 必须缓存 + 版本戳。
3. skill4 闸门是字符串匹配，本 skill 的视频类型闸是语义边界 —— 回放验证比单测重要。
4. 本 skill 的判定必须被 skill6 复用 —— 所以逻辑不在本目录，在 `_judge_core`。

## 10 维护周期与增量模式（2026-11-14）

**周期不是按月**（`_过目/225`）：新增琴型 7 天内必抓 + 存量 90 天增量 + 判定器变更全量重判。

### 10.1 游标

`原始数据/mb/_cursor_bili.json`，每款记 `lastFetchedAt / nComments / nSource / videos / judgeVer / state / runs`。

**只记事实，不记判断**：该不该抓由调用方按 `--max-age-days` 决定，内核不替业务定阈值。
`nSource` 是源里该款的原始评论条数 —— 没有它，「今天刚判过、明天来了 20 条新评论」会被误判成「未到期 → 复用」。

### 10.2 四种到期原因

| 原因 | 含义 |
|---|---|
| `no_record` | 游标里没这款（新增琴型） |
| `never_succeeded` | 从没抓成功过（n=0） |
| `stale_Nd` | 距上次 > `--max-age-days`（默认 90） |
| `source_changed_A_to_B` | 源评论数变了（**即使今天刚判过也会重判**） |

未到期的款直接复用上次产物记录（标 `reused=True` + `reusedAgeDays`），不重判、不花 token。

### 10.3 逐条标签池（增量只判新增）

`原始数据/mb/_judge_cache/_pool_bili.jsonl`，每行 `{h, v: judgeVer, l: label, at}`。
判定前先查池，**只把池里没有的评论送 API**。

```
LTD|EC-1000 源从 12 条变 14 条
→ 送判 2 条、复用 12 条、L_ai 59.09 → 66.67     （实测 2026-11-14）
```

池**只用于增量模式**：整批判定的上下文效应保证「同输入同输出」，逐条复用不能，
所以全量模式仍整批判，历史可比性不受影响。

### 10.4 命令

```bash
# 增量（推荐，日常用这个）
python skills/05_bili_l/run.py --judge --incremental

# 只跑指定款 / 改阈值
python skills/05_bili_l/run.py --judge --incremental --keys "LTD|EC-1000" --max-age-days 30

# 全量重判（改了口径或判定器才用）
python skills/05_bili_l/run.py --judge
```

增量模式下 `workers` 会被强制降到 1（标签池不跨线程共享），实测 117 款全量复用 **0.76 秒**。

### 10.5 维护步骤

1. 到期款先 `--collect --keys` 补新评论（搜索 + 抓取仍是全量抓，B站一款只有 1 个视频）
2. `--judge --incremental`：到期款重判（只送新增），其余复用
3. `--reconcile` 出 patch → skill7 `--apply-l`
4. 若 `judgeVer` 变了，先跑 `--all` 看金标准是否还过；池里旧版本标签不会被复用

## 11 封装与验收（v1.0）

- 内核自检 37 项全过；本 skill 金标准 20 款回归全过；契约 A1–A12 全过
- `prompt_sha256` 与历史 B站判定器一致（`29d8bd91…`）→ V0 等价性有机器判据
- 明文密钥 0 处（A12）

---

## 12 大杂烩榜单评论同权并入（v1.1，2026-11-14）

口径见 `Methodology §1.1.0e`。**大杂烩评论与单款测评视频评论完全同权**：
同判定器、同分母、同 `L_ai` 公式、同 `w_L`；不设 `src_w`、不打 `thin_l` 特判、不进 `brand_macro_L`。

### 12.1 两段管道

```
原始数据/mix/mix_comments_raw.json   (榜单视频评论，4,230 条)
        │  mix_source.py：品牌/型号粗筛（19.3%）→ AI 归属
        ▼
原始数据/mb/_mix_attr_<date>.json    {keys: {琴型key: [{text, rpid, bvid, polarity}]}}
        │  judge.py --with-mix：追加到该款语料池末尾
        ▼
_meta.mix.enabled = true，每款记录多出 n_mix / n_video / mixVideos
```

归属内核在 `_judge_core/attr.py`（`attr-1.0.1`），**只判「这句话在说哪把琴」，不判极性**。
极性一律交主判定器，避免两套定义打架。

### 12.2 三条硬约束

1. **顺序**：单款视频评论在前、大杂烩条目在后。规范序靠位置稳定，掺中间会改分批切分、标签池复用不上。
2. **游标**：`_n_source()` 已把 mix 条目计入 `nSource`。新榜单视频带来新条目时，「未到期」不是复用理由。
3. **契约 A14**：`n_mix` 与 `keys_hit` 必须双向一致；`mix.enabled=false` 时全场 `n_mix` 必须为 0
   ——这条专门防「接入被静默绕过」。

### 12.3 命令

```bash
python skills/05_bili_l/mix_source.py                              # 归属（走缓存）
python skills/05_bili_l/run.py --judge --incremental --with-mix     # 并入判定（只送新增条目）
python skills/05_bili_l/run.py --contract                           # A14
```

改归属 prompt 必须 bump `attr.VERSION`，否则缓存不失效，会一直返回旧归属。

### 12.4 实测（2026-11-14）

4,230 → 粗筛 816 → 归属 152 条 / 36 款；并入后 36 款 B站 `ΔL` 全部为正（中位 +2.62，max +27.59）；
折进 M 后 `|ΔM|max 0.083`，名次动 6 款最多 2 位，Top10 零变动。
详见 `_过目/228`。
