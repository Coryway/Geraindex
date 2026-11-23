# skill 07 · 合并算分导出（主榜唯一写入口）

> 版本 v1.0 · 2026-11-07 · 依据 `_过目/归档/148_skill封装规范` 七段契约
> 依赖：`抓取脚本/_pool_e_merge_20261029.py`（E/n 唯一来源）、`_apply_v515_20261106.py`（算分算式来源）、`_export_delivery_20261104.py`（导出）
> **本 skill 不联网、不采集、不写池**；只读池与主榜，唯一可写 `MultiBrand_Ranking_Data.json`。

---

## 1 输入契约

| 输入 | 路径 | 字段 | 缺失怎么办 |
|---|---|---|---|
| 主榜 | `MultiBrand_Ranking_Data.json` | `items[].price/ value.u_breakdown / retention / retentionCap / n_personal / n_channel / s_basis / p_is_proxy` | `P/E/S` 任一缺失 → 该款进 `errs`（kind=`no_input`），不静默跳过 |
| 闲鱼池 | `原始数据/mb/mb_pool_xy.json` | 逐 key `candidates[]` | 必须过 `apply_pool_patches()`；条数不守恒直接 assert 失败 |
| 别名 | `原始数据/mb/mb_uid_alias.json` | key → 池 key 列表 | 取不到池 key → `errs(kind=pool_lost)` |
| 海外 S | `mb_pool_p_os_20261102.json` / `mb_pool_rev.json` | `median_cny` | 只用于 L2 告警，不参与 L1 算分 |
| ref | `build_specs_ref/build_models_ref/build_mainP/load_ref_override` | 一手参考价 | 取不到 ref 时回退 `P`，并记 `no_ref` |

## 2 输出契约

| 产物 | 路径 | 幂等 |
|---|---|---|
| 运行报告 | `reports/_skill7_<mode>_<ts>.json` | 每次新建，不覆盖历史 |
| 主榜（仅 `--apply`） | `MultiBrand_Ranking_Data.json` | 全量重写，写后 4 条断言 |
| 影子（仅 `--mode=new`） | `_shadow/skill7_new_<ts>.json` | 不写主榜、不排名 |

## 3 风控参数

无网络请求，不占 `net_governor` 任何档位。可与任何采集 skill 并行。

## 4 闸门分层（关键：L1 判死，L2 只报）

| 层 | 内容 | 处理 |
|---|---|---|
| **L1 算分复现** | 以主榜落盘 `P/E/S/W/cap/n` 为输入，重跑 U→R→V→O→T→rank→MAX(O) | 任一维超差 = **exit 2，禁止写盘** |
| **L2 数据重算** | `merge_one` 从池重算 E/n；cap 按规则重新派生；S 与池中位比对；`price.E` 与 `ub.E` 双写比对 | **只出清单，不写盘**；`--apply` 时须 `--accept-e-diff` 显式接受 |

L2 的清单分四类，不许混成一堆：
- **待确认**（`e_diff`）：池重算与落盘不一致 → 默认阻塞 `--apply`。
- **已裁定**（`e_settled`，登记在 `run.py` 的 `E_DIFF_SETTLED`）：用户拍板后不再当新问题，但**必须写理由**。
- **e_gap 沿用**（`e_gap_kept`）：池无合格样本，按 n=0 规则留旧值。
- **告警**（`s_pool_conflict` / `dual_mismatch`）：S 归 skill 4；双写不一致必须为 0。

**ref 同源铁律**：算 pm（渠道污染判定）用的 ref 必须与算 E 的 ref 同源。历史上 `_apply_v515` 用
`p_list_median or jdAfterCouponCNY`，两者皆 None 时 merge 走 `ref=None` 分支 → pm 与 E 出自两个 ref
（EC-1000：E=5,880/pm=6,000，vs E=5,599/pm=4,995）→ dev 2.0% 误判成 12.1% → cap 0.60 应为 0.75。

为什么分开：E/n/cap 依赖池（数据），池每次采集都会变；算分链只依赖公式。混在一起会让「口径没变但数据更新了」被误判成「代码错了」。

## 5 自校验

L1 逐款 12 维对照落盘值（容差）：

| 维 | 容差 | 维 | 容差 |
|---|---|---|---|
| E | 0.5 | R | 1e-3 |
| S | 0.5 | V | `max(0.5, P×1e-4)` |
| S_adj | 0.5 | M | 0 |
| W | 0 | disc | 0 |
| U | 1.0 | n_personal / n_channel | 0 |
| T | 0.02 | MAX(O) | 5e-4（存 4 位小数的舍入残差） |

落盘前 4 条（照 `_apply_v515` 尾部）：条数守恒 / maxO 自洽 / rank 连续 / T 与 rank 单调。

## 6 已知坑

1. **不要用「简化公式」当校验基准**（E15）：U 必须带 `W`，`U=(E×W)×0.7+S_adj×0.3`。
2. **钳制必须在所有重算分支之后**（E22：n=2 两点中点曾把 0.8×ref 封顶覆盖回去）。
3. **入口必须与 `_pool_e_merge.main()` 同源**（E23）：漏 `apply_pool_patches` 会报假差异（TG-530/AZES31）。
4. **字符串匹配去空白**（E24）：闲鱼标题插空格，`cust om shop` 匹配不到。
5. **`price.usedMedianCNY` 与 `value.u_breakdown.E` 是两处写**（handoff R1）：只改一处会出现「展示 E 变了、算分 E 没变」。本 skill 已把双写不一致列为 L2 告警。
6. **`retentionCap` 才是守卫读的字段**，`retention.capApplied` 只给展示；`value.retention_chain.r_applied` 又被 `_guard_chain` 读——三处必须同源。

## 7 失败处理

| kind | 含义 | 处理 |
|---|---|---|
| `no_input` | 主榜缺 P/E/S | 记 errs，该款不参与 L1，报告计数 |
| `pool_lost` | 池里找不到该款任何 key | errs 计数（对应铁律 5，禁止 continue） |
| `parse_fail` | `merge_one` 抛异常 | errs 计数，附带异常摘要 |
| `no_ref` | 无 ref 也无可回退 P | errs 计数 |
| `true_no_sample` | 池有该 key 但无合格样本 | 进 `e_diff`，标「一侧为空」；按 n=0 规则沿用旧值 |

## 8 两条链路

| 环节 | `--mode=incr` | `--mode=new` |
|---|---|---|
| 入口 | 主榜 94 款正式 | `--input=<items.json>` |
| uid | 不变 | 须先登记 `BRAND_CODE` 发号 |
| 排名 | 参与 | **不排名**（`newPendingReview=true`） |
| 落盘 | 写主榜（须 `--confirm=APPLY-<n>`） | 只写 `_shadow/` |

## 9 回归判据（本轮已达成）

| 指标 | 目标 | 实测 |
|---|---|---|
| L1 款数 | 94 | 94 |
| 12 维失配 | 0 | 0 |
| MAX(O) | 246.8187 | 246.8187 |
| rank 与 O 降序 | 完全一致 | 一致 |
| L2 待确认 | 0 | 0（e_gap 2 款沿用、已裁定 1 款） |
| 双写不一致 | 0 | 0（STE-PRO 已于 2026-11-07 对齐为 6,873.6） |

## 10 用法

```bash
PY="python"
$PY run.py --verify                          # 只读体检（默认动作，收工前必跑）
$PY run.py --apply --confirm=APPLY-94        # 落盘（L1 全绿才允许）
$PY run.py --mode=new --input=new_items.json # 新琴影子
```
