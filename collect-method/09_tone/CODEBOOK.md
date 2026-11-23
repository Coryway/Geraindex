# tone 五轴编码手册（CODEBOOK v1.0）

> 冻结日期：2026-11-15。**编码前冻结，编码后不得改动。**
> 若某条规则在实际语料里无法执行，只能记为 `codebook_defect`，不许临时改规则再重编。

---

## 0. 为什么必须有这本手册

DS 上一轮的原话：**「真正决定会不会造数的不是轴名，而是编码手册和盲编码。
没有手册，五轴比单轴更容易造数，因为你给了自己五个自由度。」**

单轴只有一个自由度（高/低），五轴有五个。所以五轴反而更危险。
这本手册的作用是：把「什么句子能进哪条轴、算哪个方向」提前写死，
让任何人都能用同一套规则复现同一结果。

---

## 1. 五轴定义（端点已改为中性表述）

| 键 | 中文 | 0 端（低） | 100 端（高） | 只测什么 | 明确不测什么 |
|---|---|---|---|---|---|
| `low_end` | 低频量感 | 低频量少 | 低频量多 | 低频**有多少** | 下潜深度、紧/松（另记备注） |
| `high_end` | 高频突出度 | 高频不突出 | 高频突出 | 高频**冒不冒** | 亮/刺的判断（另记极性备注） |
| `mid_focus` | 中频突出度 | 中频凹陷 | 中频凸出 | 中频**顶不顶** | 中频好听与否 |
| `sustain` | 延音 | 短 | 长 | 音符**能挂多久** | 只在限定条件下采（见 §4） |
| `compression` | 压缩感 | 压缩感低（动态开放） | 压缩感高（被压扁） | 动态**被压多少** | 粗糙度（降为备注，不成轴） |

### 1.1 相比 v0.1 的三处修改（DS 指出）

| v0.1 | v1.0 | 原因 |
|---|---|---|
| `grain`（颗粒/压缩） | **拆开**：只留 `compression`，"粗糙"降为备注 | 粗糙度与压缩感不同步，一轴装两件事 |
| 端点用「瘦/刺/顶/紧」 | 改为「量少/突出度低/凹陷/压缩感低」 | 原词自带优劣，嘴上说不判、端点已在判 |
| `low_end` 混量感+下潜 | **只测量感**，下潜记 `depth_note` 备注 | 两个概念，混一起不可比 |

---

## 2. 正负锚例（每轴各 5 句，全部来自真实测评原文）

编码时先拿这 5 句校准，方向判错的先修自己的判断再编。

### 2.1 `low_end`（低频量感）

| 方向 | 原文 | 来源 |
|:--:|---|---|
| **低** | `there isn't a huge amount of low end` | 典型测评用语 |
| **低** | `偏薄但干净` | smzdm 同价位横评（Squier CV） |
| **低** | `低频收得比较紧` | 中文测评常见表述 |
| **高** | `firm, bouncy lows` | GuitarPlayer, Player II Modified Strat |
| **高** | `低频结实` / `声音体积感更强` | smzdm（Shijie TM4 / Farida） |

### 2.2 `high_end`（高频突出度）

| 方向 | 原文 | 来源 |
|:--:|---|---|
| **低** | `the neck delivers a lovely and warm tone` | Guitar World, Player II Strat |
| **低** | `音色柔和高频也不刺耳` | smzdm, Shijie TM4 |
| **低** | `只是高频略收敛` | smzdm 横评, Farida |
| **高** | `an ultra-bright bridge pickup` | Guitar World, Player II Tele |
| **高** | `a boatload of treble` | GuitarPlayer, Player II Modified |

### 2.3 `mid_focus`（中频突出度）

| 方向 | 原文 | 来源 |
|:--:|---|---|
| **低** | `classically scooped midrange` | GuitarPlayer, Player II Modified |
| **低** | `中频凹` | 中文测评常见 |
| **低** | `scooped mids` | 通用 |
| **高** | `nice midrange bite` | Guitar World |
| **高** | `中频顶出来` / `中频厚` | 中文测评常见 |

### 2.4 `sustain`

| 方向 | 原文 | 来源 |
|:--:|---|---|
| **低** | `Less sustain than a Les Paul` | Guitar World, RG550 |
| **低** | `音符衰减快` | 中文测评常见 |
| **低** | `notes die off quickly` | 通用 |
| **高** | `It sustained well even played unplugged` | GuitarPlayer, Player II Modified |
| **高** | `延音很长` / `挂得住` | 中文测评常见 |

### 2.5 `compression`（压缩感）

| 方向 | 原文 | 来源 |
|:--:|---|---|
| **低** | `动态强` / `清晰动态强` | smzdm, Shijie TM4 |
| **低** | `open and dynamic` | 通用 |
| **低** | `力度反馈清晰` | smzdm |
| **高** | `offers a hint of compression at higher gain settings` | MusicRadar, RG550（⚠ gain_dependent） |
| **高** | `动态压缩明显，细节糊成一片` | smzdm 横评（Fender Player） |

---

## 3. 纳入 / 排除规则

### 3.1 纳入（必须同时满足）

1. 句子描述的是**声音特征**，不是外观、价格、做工、手感、配置清单
2. 能定位到**具体型号**（按 §5 归属规则）
3. 句子里的描述词能映射到 §1 五轴中的**至少一个**，且方向明确

### 3.2 排除（命中任一条即弃）

| 排除类型 | 例 | 原因 |
|---|---|---|
| 纯泛评 | `音色很好` / `sounds incredible` / `sounds amazing` | 无方向，不可编码 |
| 价格档位 | `这个价位少有的好声音` | 价格信号，正是要剥离的 |
| 外观 | `巨帅巨亮眼` / `dark night telecaster` | `亮`/`dark` 指漆色 |
| 配置清单 | `Alnico V single coils, 9.5" radius` | 是事实不是听感 |
| 手感 | `琴颈很顺` / `neck feels great` | 属 feel 四轴，不属 tone |
| 工艺/QC | `搓品丝` / `nut not cut properly` | 属 reliability |

### 3.3 夸张词单列

`screaming` / `brutal` / `insane` / `killer` / `炸裂` / `天花板` → 记 `hyperbole=true`，
**仍可编码方向**（如 screaming highs → high_end 高），但在聚合时降权 0.5。

---

## 4. 设备依赖处理（`sustain` 尤其关键）

DS 指出：`sustain` 若不分测试条件，带高增益的延音句与清音延音句不可比。

| 语境标记 | 判定 | 处理 |
|---|---|---|
| 明确写 `unplugged` / `不插电` / `acoustic volume` | `clean_context` | 正常计入 |
| 明确写 `clean` / `清音` / `clean channel` | `clean_context` | 正常计入 |
| 明确写 `high gain` / `高增益` / `cranked` / `overdrive` | `gain_dependent` | **`sustain` 轴不计入**；其余轴可计入 |
| 未写清语境 | `unknown_context` | `sustain` 轴不计入；其余轴计入 |

**其余四轴（low_end / high_end / mid_focus / compression）不受此限制**，
但若整句只在高增益语境下成立（如 `at higher gain settings`），记 `gain_dependent=true` 并在聚合时降权 0.7。

---

## 5. 句子归属规则（T2 一页多琴时）

沿用 skill 08 的三级校验：

- **L1 段落自证**：小标题 / 段落首句明确写出型号 → 采用
- **L2 窗口唯一**：句子前后 200 字内出现**唯一**型号名 → 采用
- **L3 价格交叉**：同段提到价格，与主榜 P 相差 ≤20% → 采用
- 三条都不满足 → **丢弃**，记 `unassigned`，不猜

**归属等级记入每条证据的 `assign_level`（L1/L2/L3），供后续敏感性分析。**

---

## 6. 盲编码要求

- 词典法本身**不看**型号、品牌、价格，只按词匹配 → 天然盲
- 人工/LLM 复核时**先隐藏型号与价格**，只看句子
- 双编码（gate 1）：两份独立编码，算 Cohen's κ
  - 编码 A：规则词典法
  - 编码 B：LLM 独立判读（不给轴定义以外的任何信息）
  - **κ ≥ 0.6 才继续**；<0.6 说明手册歧义大，回炉重写 §1–§3

---

## 7. 已知无法处理的句子类型（诚实列出）

1. **复合句**：`warm but bright` 同时指向 high_end 的两端 → 拆成两条相反证据，各记 1 票，聚合时相互抵消（等价于"该句无信息"）
2. **比较句无基准**：`warmer than expected` 不给基准 → 丢弃
3. **风格映射**：`great for jazz` 不描述声音 → 丢弃（风格不是声学属性）
4. **乐器对比的"更好"**：`better than a Les Paul` 无具体维度 → 丢弃
5. **翻译损耗**：中文测评的"通透""干净"等词跨语境含义不稳 → 只收 §2 锚例中已确认的用法，
   新词首次出现时记 `new_term` 并人工确认后才纳入

---

## 8. 版本与变更纪律

- 本文件 v1.0 自编码开始即冻结
- 任何后续变更必须**另起 v2.0 并重编全部语料**，不许在 v1.0 上打补丁后继续用
- 变更原因写进 `_过目/` 对应报告，附旧版对比
