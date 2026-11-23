# skill 06 · YouTube 评论旁证（L）— v1.0（2026-11-14 封装）

YouTube 评论 → T/W/P/N → `L_ai`。判定与 skill5 **共用** `skills/_judge_core/`，
同一份分类定义、同一个 parse、同一个 L 公式；差异只在语种提示与灌水规则（受控参数块）。

---

## 1 输入契约

| 输入 | 位置 | 说明 |
|---|---|---|
| 采集产物 | `原始数据/mb/mb_c*_yt*.json`（config 的 `sources` + `extra_sources`） | `{yt_key: {videos:[{videoId,title,comments:[{text,votes}]}]}}` |
| 注册表 | `skills/06_yt_l/registry.json` | 154 个 YT key → 主榜 key 映射（命中 109） |
| 源指纹 | `skills/06_yt_l/_source_fingerprint.json` | 13 个源文件的 sha256，**变了就 exit 2** |

**源清单顺序不可改**：87 个 key 跨多个文件重复，换顺序会换评论集，等价性基准就没了。

## 2 输出契约

`原始数据/mb/_yt_l_<YYYYMMDD>.json`，`records` 以 **yt_key** 为键（长名），另带 `board_key` 与 `group`。

对账时（`reconcile.py`）按主榜 key 合并，合并规则与 `_apply_LM` 一致：
**counts 相加后重算 L_ai**，不是加权平均。4 个合并款见 `MERGE_MAP`。

## 3 采集参数

| 参数 | 值 | 说明 |
|---|---|---|
| 视频/款 | ≈8 | 历史采集结果，不动 |
| 评论/视频 | 120 | `youtube-comment-downloader` 直连，不走代理 |
| 采样 | **none** | 历史判定未采样（n 中位 196、max 389），去重即全集 |
| 最低播放量 | 1 万 | 低于此值的视频评论噪声大 |

## 4 三道闸

- **L1 采集**：搜索走 CDP（`yt_search.js`），评论走 `youtube-comment-downloader`。
  搜索结果的推荐序会漂移，所以 `--collect` 要求显式 `--ids-file`（videoId 一旦确定就是确定输入）。
- **L2 视频类型闸**：剔 翻弹/开箱/二手/改装/教学；收 测评/对比/复刻/长期；未知保留，视频级报警 >15%。
- **L3 契约**：A1–A12，A11 为源指纹阻断。

## 5 自校验

```bash
python skills/06_yt_l/run.py --selftest      # 20 个 YT key 金标准
python skills/06_yt_l/run.py --contract      # A1–A12
python skills/06_yt_l/run.py --all
```

## 6 已知坑

| 坑 | 现象 | 处置 |
|---|---|---|
| 源顺序依赖 | 同款两次跑出不同评论集 | 源清单顺序写死 + 指纹进产物（A11 阻断）。**顺序即规范序**，不再按内容哈希重排（哈希重排会改分批，实测让 ΔL 中位漂到 1.0 条） |
| API 非确定 | temp=0 下个别款差 1–2 条评论 | 靠缓存冻结；验收用「噪声底」相对判据，不设绝对阈值 |
| key 长名不对齐 | 对账时主榜款拿不到 L | `registry.py` 三级匹配：精确 → `|` 后段 → 以主榜 key 结尾的最长匹配 |
| 合并款 | AZ2203/2204 等拆成两条 | `MERGE_MAP` 合并 counts 后重算 L |
| 评论语言 | 西/葡/泰/韩语被判 N | prompt 词例参数块已覆盖，不许因看不懂判 N |
| 广告评论 | 「free guide」「check my channel」 | 判 N；灌水规则已在参数块 |
| 播放量低的视频 | 评论极端 | 采集闸 1 万播放 |

## 7 失败处理

单款失败 → `judge_failed`，不整批重试；`miss` 必写原因；`no_video` 不入对账。

## 8 命令

```bash
cd <项目根>

python skills/06_yt_l/run.py --registry
python skills/06_yt_l/run.py --judge                    # 全量（有缓存则快）
python skills/06_yt_l/run.py --judge --no-gate          # 等价性验证专用
python skills/06_yt_l/run.py --reconcile                # 出 L patch（按主榜 key）
python skills/06_yt_l/run.py --collect --ids-file ids.json
python skills/06_yt_l/run.py --accept-sources           # 显式接受源清单变更
python skills/06_yt_l/run.py --all
```

## 9 与 skill5 的差异

| 维度 | skill5（B站） | skill6（YT） |
|---|---|---|
| 源 | 单文件 | 13 个文件，顺序固定 |
| key | 即主榜 key | 长名，需映射 + 合并 |
| 采集 | CDP + 登录态 | 直连 API，无需登录 |
| 闸门报警 | 款级（1 视频/款） | 视频级（约 8 视频/款） |
| 权重 | `src_w=1.0` | `src_w=0.45`（读 `rank_params.json`，不写死） |

## 10 维护周期与增量模式（2026-11-14）

周期同 skill5：新增 7 天必抓 + 存量 90 天增量 + 判定器变更全量重判。游标 `_cursor_yt.json`（按 yt_key）。

**YT 与 B站的关键差别**：B站一款只有 1 个视频，评论只会增长；YT 一款约 8 个视频，**同型号会不断出现新测评**。
所以 YT 的季度任务必须先「扫视频列表」找出新视频（整视频抓），再抓老视频的评论增量 —— 只抓老视频会漏新测评。

```bash
python skills/06_yt_l/run.py --judge --incremental            # 到期款重判，其余复用
python skills/06_yt_l/run.py --judge --incremental --max-age-days 30
```

维护步骤：
1. 新视频 → `--ids-file` 补采 → 合并进 `sources`（改了要 `--accept-sources` 并写记录）
2. `--judge --incremental` → `--reconcile` → skill7 `--apply-l`
3. 源文件一旦变动，A11 会先拦截；确认是「有意更新」才 accept

游标与池的实现见 skill5 §10.1–10.3（共用 `_judge_core/cursor.py` 与 `label_pool.py`）。

## 11 封装与验收（v1.0）

- 内核自检 37 项全过（与 skill5 同一套）；本 skill 金标准 20 key 回归全过
- `prompt_sha256` 与历史 YT 判定器一致（`c63e4a10…`）
- 源指纹已冻结；主榜 47 个 yt 款全部能映射（0 缺口）
