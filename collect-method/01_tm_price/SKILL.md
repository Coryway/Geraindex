# skill 01 · 淘宝一手价（P）— v2.4

> 2026-11-08 · 按 `_过目/归档/148_skill封装规范` 七段 + 双链路封装
> **采集内核不重写**：一律调生产脚本 `抓取脚本/_tm_batch_20261031.py`。
> 本 skill 负责：入口（`collect.py`）、闸门（`run.py`）、契约（`contract.py`）、自检（`selftest.py`）。
>
> **v2.3 三处修正（2026-11-09 用户拍板）**
> ① **套装闸只认贵重物**：`RE_SKU_BUNDLE` 不再用「套装/礼包/套餐/plus版/加购」这类空词，
>    只认「音箱/效果器/功放/声卡/踏板/单块/监听/调音器」。**琴包/连接线/琴架/拨片/背带是出厂赠送，
>    不加价、不许因此剔样本**（淘汰旧 v2.2 第 ① 条「只留套装/礼包…」的写法）。
>    实证：赛文琴行 item 641398043607 一页 60 SKU，`【标配】琴+全套配件`（裸琴价）与
>    `【全家桶】+JAM 效果器音箱`（加价）并列 —— 空词分不开，「音箱/效果器」一眼分开。
>    `RE_SET`（记录级）同步删「整套/全套」。
> ② **兄弟型号纯字母名也判串款**：新增 `_sibling_hit()`。`Les Paul Junior` 被
>    「LP Special … SG/LP Junior」合集页冒充（参数面板写 `special`，无数字无连字符，
>    旧判据一律放行）——2 条 3,000 / 2,046 已正确剔除。
> ③ **琴体排除词必须写在 registry**：淘宝侧 ex 原先只有 8 个通用词，没琴体 →
>    一个 `affinity` 收进 Strat/Tele/JM/Bass（与闲鱼 `EXCL_OVERRIDE` 不对齐）。
>    Squier 6 款已补，重放剔掉 5 条真串款。
>
> **v2.2 五处修正（2026-11-08，62 款补采暴露的误杀，逐条有款有钱数）**
> ① **SKU 套装闸误杀赠品**：`RE_SKU_BUNDLE` 原含「琴包」→ SG 旗舰 4 条 C1（3,600×3/3,400）全灭。
>    删掉琴包/背包/连接线/拨片/背带（赠品不加价）——**判据后来在 v2.3 收紧为只认贵重物**。
> ② **合集页参数面板被当串款证据**：一页 18 个 SKU 时参数面板只写基础款（`MA-3`），
>    与 must `ma3pro` 不符 → MA3Pro 17 条全灭。改为：SKU 虚拟记录的参数判据用 **SKU 文本**；
>    判据从「cp 以 must 开头」改「cp **包含** must」（SKU 带 `【爆款】` 前缀时 startswith 必败）。
> ③ **`KS/KT100` 共享数字合集漏网**：新增 `RE_SLASH_SERIES`（两字母系列 + `/` + 字母系列 + 数字）。
> ④ **SKU 加价套装**：新增 `RE_SKU_PLUS`（`+` 后跟 ≥2 英文字母，排除 VIP/赠/送/配/包/线/架/拨片/背带）。
> ⑤ **粘连型号放行**：`_near_model`（相似度 ≥0.85 且长度差 ≤2），救回 `MA3PPROS`=MA3 PRO S；
>    实测 `az2204/az2203`=0.833、`rss20/rse20`=0.80 仍拦。
>
> **v2.1 三处修正（2026-11-08，月度维护实测缺陷）**
> ① `run.py --layers --all` 改**款级取最新采样**（原按批次日分组 → 月度只补 10 款时其余 84 款全判「无基准」）；旧行为保留为 `--by-day`。
> ② 新增**采集产物注册表** `ARTIFACT_PATTERNS`（`_tm_live_B*` / `_tm_c1fill_*` / `_tm_epi_*` / `_tm_punify_*` …）＋ `--artifacts [--audit]`：补采产物不再「采了不用」（E16 第 15 例）。
> ③ `collect.py` 的 `P_paid(实付)` 不再默认入选款理由（内核取不到结算页实付，会白烧额度），改用 `--with-paid` 显式触发并转结算页脚本。

```
skills/01_tm_price/
├── SKILL.md        七段契约（本文件）
├── collect.py      采集入口：--mode=incr|new，产队列 → 调内核 → 契约 → 影子/入池
├── run.py          闸门 + 账层：--replay / --layers / --all / --artifacts
├── contract.py     输出契约 + A1–A4 自校验
├── selftest.py     全产物回归（v2.2 起覆盖 95 款 / 426 样本，--freeze 冻基线）
└── _selftest_baseline.json   冻结的判定基线
```

---

## 1 输入契约

| 输入 | 路径 | 关键字段 | 缺失怎么办 |
|---|---|---|---|
| 搜索词注册表 | `原始数据/mb/_tm_query_registry_20261106.json` | `key/uid/brand/query[]/must[]/ex[]/allow[]` | **exit 2**，不许凭空猜词 |
| 品牌型号表 | `抓取脚本/mb_models.json` | `brands.<品牌>.models[].tokens/deny/allow` | `--mode=new` 时品牌不在表内 → **exit 2** |
| 主榜（只读） | `MultiBrand_Ranking_Data.json` | `price.jdAfterCouponCNY`(＝P) / `P_proxy` / `P_paid` / `P_list_n` | 缺则按「P 缺失」排队 |
| 采集产物 | `原始数据/mb/_tm_*.json` | 每条一个详情页，见 §2 | 缺文件 → 记 `no_batch` |

**P 的定义（2026-11-08 定案）**：`P = P_list`（**优惠前**，同款有 C1 时只取 C1 中位）。
`P_paid`（实付）**只记录、不进公式**，取值优先结算页「立即支付￥X」，其次页面「店铺优惠后」。

---

## 2 输出契约

采集产物（内核直出，本 skill 不改结构）：

```json
{"<key>": [ {"itemId","title","shop","beforeCoupon","afterShopCoupon","firstYen",
             "paramModel","bodyLen","priceSrc","sku","skuN","isFrom","captcha","noitem"} ]}
```

`contract.py` 规整后的标准契约（148 §三）：

```json
{"<key>": {
  "key","uid","query",
  "samples":[ {"itemId","title","shop","list_price","after_coupon","first_yen",
               "param_model","bodyLen","c1","price_src","sku"} ],
  "no_price":[ {"itemId","reason_code","note"} ],
  "drop":{"ex_hit":3,"collection":2,"dup":1},
  "n_detail":7, "n_card":35
}}
```

**幂等**：产物文件按 `_skill1_<mode>_<ts>.json` 命名，**不覆盖历史**；
同一款重跑由内核的 `store` 断点续采（已完成的款自动跳过，`TM_FORCE=1` 才重采）。

**采集产物注册表（v2.1）**：能进账层的文件模式只有三个——`_tm_live_B*_*.json`、`_tm_c1fill_*.json`、`_tm_epi_*.json`。
新增一类补采产物（如 `_tm_xxx_*.json`）**必须加进 `run.py::ARTIFACT_PATTERNS`**，否则账层看不见它，等于白采。
`run.py --artifacts --audit` 会逐文件列出「款级最新来源」，并对「有价但主榜四个落地字段全空」的款 **exit 2**。

---

## 3 风控参数（不可放宽）

| 操作 | kind | gap | burst | 说明 |
|---|---|---|---|---|
| 搜索页 | `tm_search` | 150–180s | 2 次后 600–900s | 全流程瓶颈 |
| 详情页 | `tm_detail` | 90–120s | 12 次后 900–1500s | 阶段 2（整夜零熔断后启用） |

- **并行禁忌**：淘宝 + 闲鱼 **禁止**并行（同一 IP 同域叠加必封）；京东 + 闲鱼可以。
- **熔断**：页面命中 `验证/安全/login.taobao/punish/x5sec/captcha` → 写 `抓取脚本/_tm_halt.flag` 并 `exit 3`，
  **不重试、不换档**。见到 flag 立刻停手，等人工过验证。
- CDP 端口 9225（`tmall-profile`），由 `_cdp_keeper.py` 每 20s 巡检自愈。

---

## 4 三道闸（L1 采集时 / L2 合并时 / L3 契约）

| 层 | 在哪 | 判什么 |
|---|---|---|
| **L1 采集时**（`_tm_batch_20261031.py`，省额度） | 卡片层 | `need`（词间 OR、词内 AND）、`deny`、`anthology` 合集（两个带数字型号代码并列）、品牌一致性、相对价格下限（中位×0.35）、订金/预售、C1 优先排序 |
| **L1 详情层** | 详情页 | 参数面板型号、`isFrom`（「￥X 起」不可作单价）、`skuN≥3` 触发逐 SKU 展开（仅 C1）、`captcha/loginWall/noitem` |
| **L2 合并时**（`run.py::gate`，顺序即优先级） | 见下表 | 全部 12 道 |
| **L3 契约**（`contract.py`） | 落盘前 | A1–A4 四条断言 |

`run.py::gate` 13 道闸：

| # | 闸 | 判据 |
|---|---|---|
| 0 | `vision_<verdict>` | **视觉闸 0g（2026-11-09 接入）**：`原始数据/tm/vision_2026/verdicts.json` 里该 itemId 判非 `ok` → 出池。只认 `ok`，**没判过的照旧放行**（不做「未判即剔」），`TM_VISION=0` 可关。抓的是文本看不出的：Squier 冒充 Fender、套装图、二手翻新、同页混卖另一把 |
| 1 | `blocked` | `noitem / captcha / loginWall` |
| 2 | `wanted` | 求购 / 收购 |
| 3 | `ex_hit` | 注册表 `ex[]` 命中 |
| 4 | `collection` | 斜杠并列 / 连续裸数字 / 同标题 ≥2 个型号 token（字母位 `{1,2}`） |
| 5 | `grey` | 代购 / `X折` / 通利 / 港版 |
| 6 | `bundle` | 含/带/送**音箱·效果器·功放·声卡·踏板**（2026-11-09 起只认贵重物；「套装/礼包/整套/全套」**不再**作判据）/ 瑕疵·特价处理 |
| 7 | `must_miss` | `must` **词间 OR、词内 AND**（与采集内核同源） |
| 8 | `brand_mismatch` | 目标品牌词全不出现 + 出现**他牌**词（标题只写型号 → 放行） |
| 9 | `signature` / `lefty` | 签名款（目标型号本身是签名款则放行）/ 左手款 |
| 10 | `param_mismatch` | 参数型号与 must/allow 无交集；**剥品牌词**再比，只允许 `pm = must + 后缀`；含 `-` 且 ≥6 字符的完整型号名即便无数字也算反证；**同族兄弟型号纯字母名（special/studio/standard/custom/junior/traditional/tribute/faded/deluxe）也判串款**（2026-11-09 加）；纯数字（货号）不算反证 |
| 11 | `price_order` | 优惠前 < 券后 |
| 12 | `no_list_price` | 优惠前缺失（节点与正文均无价）→ **显式计数剔除** |
| — | `dup` | 同标题同价 / 同店同价 |

---

## 5 自校验（落盘前必须全过）

| # | 断言 | 不过怎么办 |
|---|---|---|
| **A1** | 条数守恒：`samples + no_price + Σdrop == n_detail` | exit 2，**不许带着缺口落盘** |
| **A2** | 参数型号：sample 的 `param_model` 与 `must/allow` 有交集（剥品牌词后比）。纯数字＝货号、无数字＝系列名 → 只告警 | 有字母有数字但不匹配 → exit 2（串款） |
| **A3** | `list_price >= after_coupon` | exit 2（页面错乱） |
| **A4** | 注册表全覆盖（`--require-all` 时）：每款都要出现在 samples/no_price/drop | exit 2（凭空消失） |
| **A5** | **三方对账**：主榜有 `P_list` 的款，台账 `sources.taobao` 必须有对应记录（`reconcile.py` 零缺口） | 有缺口先查是不是「采了直写主榜没回写台账」；确认后 `reconcile.py --apply` 回填并标 `src=mainboard_backfill` |

**复现判据**（对主榜，不联网时用 `--all`）：
每款 ≥3 条优惠前才判「一致」；\|Δ\|≤3% 判一致、≤15% 判波动；偏差 >15% 先查自己的闸门（硬约束：某闸杀 >20% 先怀疑闸门）。

---

## 6 已知坑（都由实测触发）

| 坑 | 现象 | 根因/修法 |
|---|---|---|
| `need` 多词按 AND | `Squier\|CV Tele` 44 张卡过 0 张 | 变体词展平后要求全命中 → 改「词间 OR、词内 AND」 |
| 账层 must 整串紧邻 | `Affinity Tele` 5 条全判 must_miss | 只修了采集层，账层漏改 |
| 品牌同名款占满 pick | `Epiphone\|Les Paul Special` +1080% | 缺品牌一致性闸 → 目标品牌缺失且出现他牌才剔 |
| 参数面板抽到货号 | Fender `0140822590`、Jackson `2916342500` | 纯数字不当型号反证 |
| `KS650HH` 被误杀 | KS650 采到 0 有价 | must 写 `keipro ks650`，参数只写型号 → **剥品牌词**再比 |
| `Std 50S` 被判串款 | LP 采纳从 4 条掉到 1 条 | 50s/60s 是变体不是型号 → 注册表 `allow[]` 白名单 |
| `50/60S` 被斜杠合集闸杀 | 同上 | 白名单里的数字对先剔除再判合集 |
| 无券页 `优惠前=None` | 全库 73 条有首价无价 | body 里没有「优惠前」三字 → 改读 `[class*="highlightPrice"]` 节点，记 `priceSrc` |
| 「￥X 起」当单价 | SG 3,400 被当 SG Standard 价 | 一页 18 SKU 覆盖 5 型号 → `expand_sku()` 逐项点，`isFrom` 标起价 |
| 详情页跳 `confirm_order` | 白跑或读错价 | 在确认单直接取实付+划线价；取不到记 `redirect_dead` 放弃 |
| 折扣率反推 | 同款 3 SKU 折扣 72%/85%/73.6% | **禁止 `÷0.85` 反推**，券按 SKU 发 |
| 全量重采取优惠前 | 白跑 5–8 小时 | C3 确认单**没有划线价**；C1 两口径 6/6 完全相等 → 不必重采 |
| 月度按批次日期取数 | 只补 10 款时，其余 84 款在账层全变「无基准」 | `--all` 改**款级最新采样**；超 45 天标 `stale`（`--by-day` 保留旧行为） |
| 补采产物不入账 | c1fill/epi 这类新文件既不在 `_tm_live_B*` 里，账层永远看不见 | 加进 `ARTIFACT_PATTERNS` + `--artifacts --audit` exit 2 |
| **「配琴包」被当套装** | SG 旗舰 4 条 C1（3,600×3/3,400）全灭，C1 只剩 60s 的 4,100 | 赠品不加价 → 套装词删掉琴包/背包/连接线/拨片/背带 |
| **合集页参数面板当串款证据** | MA3Pro 17 条全灭，P_list n=9→0 | 一页多 SKU 时参数面板必写基础款 → SKU 虚拟记录改用 SKU 文本；判据 `startswith` 改 **`in`**（SKU 带 `【爆款】` 前缀） |
| **`KS/KT100` 共享数字合集** | KT100 采到 KS 系列价 2,080 | 新增 `RE_SLASH_SERIES`（字母系列 / 字母系列 + 数字） |
| **SKU「+AGA」加价套装** | MA3Pro 中位被抬到 2,922 | `RE_SKU_PLUS`（`+` 后跟 ≥2 英文字母；**排除的是赠品词，不是「款」**——送琴包/线/架/拨片/背带的 SKU 仍算裸琴价照常采纳） |
| **粘连型号** | `MA3PPROS`=MA3 PRO S 被判串款 | `_near_model`：相似度 ≥0.85 且长度差 ≤2（`az2204/az2203`=0.833 仍拦） |
| **长型号空格并列合集** | `AZES40 AZES31` 两个长型号判不出（`RE_MULTI_MODEL` 只认 1–2 字母前缀） | 参数面板出现 ≥2 个完整型号且含非目标型号 → `collection` |
| **自检按 itemId 比** | 一页 18 SKU 共用一个 itemId，条数增减检测不出（SG 误杀漏检） | 判定键改 `itemId\|sku`；覆盖改走 `latest_per_key()`（95 款） |

---

## 7 失败处理（禁止 `continue`）

任何采不到的样本，必须落 `no_price` 并带理由码，**不许静默跳过**：

| reason_code | 触发 | 后续 |
|---|---|---|
| `noitem` | 宝贝不存在/已下架 | 下次重试 |
| `captcha` | 验证码/滑块/登录墙（已熔断） | 等人工过验证再跑 |
| `no_list_price` | 优惠前缺失（节点与正文均无价） | 下次重试；连续 3 次则改走结算页取实付 |
| `price_is_from` | 只有「￥X 起」且无 SKU 明细 | 下次带 SKU 展开重采 |
| `redirect_dead` | 详情页反复被弹到确认单且确认单无价 | 留档不重试 |
| `sku_not_found` | 目标 SKU 在页面不存在 | 人工核型号是否改款 |

---

## 8 四条命令

```bash
PY="python"
cd "<项目根>/skills/01_tm_price"

# ① 看要采哪些款（不联网、不起浏览器）
"$PY" collect.py --mode=incr --plan-only
"$PY" collect.py --mode=incr --keys=PAC112V,RSS20 --pick=5 --plan-only

# ② 真采（会起浏览器，逐款按 GOV 节奏）
"$PY" collect.py --mode=incr --keys=PAC112V --pick=5 --run
"$PY" collect.py --mode=new --brand=Keipro --model=KS700 --query="Keipro KS700" --run   # 只写影子

# ③ 契约校验（对已有产物）
"$PY" contract.py ../../原始数据/mb/_tm_live_B1_20261107.json
"$PY" contract.py ../../原始数据/mb/_tm_live_B*.json --require-all      # 全榜必须齐
"$PY" contract.py ../../原始数据/mb/_tm_c1fill_20261107.json           # 补采产物同样要过契约

# ④ 账层 / 自检
"$PY" run.py --artifacts               # 采集产物注册表（哪个文件是哪些款的最新来源）
"$PY" run.py --artifacts --audit       # 有价但未入账 → exit 2（E16 硬闸）
"$PY" run.py --layers --all            # 全榜四层账（款级最新采样 + 与主榜逐款比对）
"$PY" run.py --layers --all --by-day   # 旧行为：只取最新一天那一组批次
"$PY" selftest.py                      # 42 款回归（判定变了就 exit 2）
"$PY" selftest.py --freeze             # 改闸门后重新冻基线（要写清为什么改）
```

---

## 9 双链路（148 §八）

| 环节 | `--mode=incr`（存量） | `--mode=new`（新琴） |
|---|---|---|
| 选款 | 注册表交集 + 自动挑「P 缺 / 优惠前样本<3」（`P_paid` 缺仅在 `--with-paid` 时计入） | `--brand/--model/--query` 直给 |
| uid | 沿用 | 品牌须已登记，否则 **exit 2** |
| 采集 | 增量补数 | 全链路首采 |
| 落盘 | 契约 + 可入池 | **契约 + 回写 uid 台账**（`sources.taobao`），**不写主榜**；另留 `_shadow_skill1_new_<ts>.json` |
| 排名 | 参与 | **不参与**，进观察区等人审 |
| 失败 | `no_price` + 理由码，下次重试 | 留档**不重试**（避免反复打风控） |

**硬约束**：`--mode=new` 永远不许写 `MultiBrand_Ranking_Data.json`；两链路共用同一份 L1 闸与 `GOV` 参数。

---

## 10 月度维护标准流程（v2.1）

> 目标：每月只补「过期 / 样本不足 / P 缺失」的款，不再全量重采 94 款。

| 步 | 命令 | 判据 |
|---|---|---|
| ① 看缺口 | `run.py --artifacts` | 每款最新来源文件是哪份；`stale`（>45 天）几款 |
| ② 定计划 | `collect.py --mode=incr --plan-only` | 列出「P 缺失 / 优惠前样本<3」的款，人工确认 |
| ③ 真采 | `collect.py --mode=incr --run` | 内核按 `GOV` 节奏跑；撞验证码写 `_tm_halt.flag` 即停 |
| ④ 入账 | `contract.py <产物> --require-all` → `run.py --artifacts --audit` | 契约 0 失败；**audit 必须 exit 0**（否则就是采了不用） |
| ⑤ 复现 | `run.py --layers --all` | 一致率不降；`异常` 逐款给理由 |
| ⑥ 实付（可选） | `--with-paid` → `_build_checkout_jobs_20261108.py` → `_tm_checkout_batch_20261107.py` | `P_paid` 只记录、**不进公式** |
| ⑦ 落榜 | **交 skill 7**（本 skill 不写主榜） | 守卫八项 + `MAX(O)` 断言 |

**新增一类补采产物时**：文件名必须带 `_YYYYMMDD`，并加进 `run.py::ARTIFACT_PATTERNS`，否则第 ④ 步看不见它。

---

## 11 封装与验收（v2.4，2026-11-10）

### 11.1 这个 skill 的边界

| 做 | 不做 |
|---|---|
| 采淘宝一手价 `P_list`(优惠前) 与 `P_paid`(实付，只记录) | **不写主榜**（交 skill 7 落榜） |
| 三方闸：品牌/型号/价格，含 SKU 套装与参数面板 | 不判「值不值」、不算保值率公式 |
| 视觉闸 0g（淘宝侧）：型号对不对、是不是裸琴零售价 | 不抓海外价（Reverb 归 skill 2/3 的 S 口径） |
| 台账回写 + 三方对账 | 不合并型号（合并口径在项目规则 11） |

### 11.2 五条命令（对外只有这五个入口）

```bash
PY="python"
cd "<项目根>/skills/01_tm_price"

$PY run.py --artifacts                 # ① 看现状：每款最新产物、stale 几款（不联网）
$PY collect.py --mode=incr --plan-only # ② 定计划：列出 P 缺失 / 样本<3 的款
$PY collect.py --mode=incr --run       # ③ 真采（起浏览器，按 GOV 节奏）
$PY contract.py <产物> --require-all   # ④ 契约校验（0 失败才准入）
$PY selftest.py                        # ⑤ 回归（基线 _selftest_baseline.json）
$PY reconcile.py [--apply]             # ⑥ 三方对账
```

### 11.3 验收清单（交付前逐条打勾）

- [ ] `contract.py` 0 失败 0 警告（A1 条数守恒）
- [ ] `selftest.py` 全过，且**基线变化逐条能解释**（`--freeze` 前先备份旧基线）
- [ ] `reconcile.py` 零缺口（A5）
- [ ] 抽 1–2 款手算中位数，与台账 `P_list` 一致
- [ ] 误杀自检：某道闸杀掉 >20% 的款，先怀疑闸门再怀疑数据

### 11.4 封装时踩过的四个坑（照抄即可避开）

| 坑 | 现象 | 根因 | 已修 |
|---|---|---|---|
| **字段名读错** | 契约明明 4 条样本，台账写成 `has=False/n=0/P=None` | 契约字段叫 `list_price`，代码读 `beforeCoupon` | collect.py 改读 `list_price` |
| **0 样本仍 exit 0** | 整批没采到任何价，进程却成功退出 | 没有「零样本即失败」的硬闸 | `n_s==0` 或抽不到价 → `EXIT4`，不写台账不落影子 |
| **采了直写主榜** | 主榜有 `P_list`，台账一条记录都没有（36 款） | 早期跑完直接改主榜，漏了 `bindsrc` 回写 | `reconcile.py` 对账 + A5 断言 |
| **提速 16 倍** | 详情页 3s/条，短时间被风控 | 为赶工期把 `domcontentloaded` 改成 `commit`，速率从 50s/条变 3s/条 | 硬性下限 12–15s/条、单日 ≤300 条 |

### 11.5 谁在用这份产物

| 下游 | 用什么 |
|---|---|
| skill 7（落榜） | `P_list`（真值 100% 权重，proxy 打 `disc=0.95`） |
| 主榜 `price.P_list` / `price.p_is_proxy` | 上面的直接映射 |
| 月度维护 | 本文件第 10 节流程 |
