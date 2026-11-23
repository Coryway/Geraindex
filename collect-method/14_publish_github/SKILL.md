# skill 14 · 一键发布到 GitHub

> 版本 v1.1 · 2026-11-23（v1.1 起区分「前端展示」与「方法论」两种发布范围）
> 入口：`python skills/14_publish_github/run.py`
> 依赖：git（`<工具路径>\git.exe` 或 `GIT_EXE` 环境变量）、已初始化的发布仓库（默认 `<发布目录>/qidu`）
> **不改主榜、不改前端交付包**；只写发布仓库与 git 记录。

---

## 人话版（先读这段）

网站上有两类东西，更新节奏完全不同：

- **前端展示**（`docs/` 里那 5 个文件）：榜单数据每月会变，所以**每月都要发**。
- **方法论**（`collect-method/` 里 20 份 md）：判读规则、公式、流程说明，**逻辑不改就不用发**。每月跟着重推一遍只会产生一堆"内容完全相同"的提交，把提交历史搞脏。

所以这个 skill 默认**只发展示**。哪个月改了方法论（比如调了音色判读档位、改了净亏损公式），再加一句 `--scope method`。

判断很简单：

| 你干了什么 | 用哪条 |
|---|---|
| 改了主榜数据、重导了 `data.js`、重打了 `tiers.js` | `run.py --tag`（默认只发展示） |
| 改了某个 `SKILL.md`、判读手册、公式说明 | `run.py --scope method` |
| 数据和方法论都改了 | `run.py --scope all --tag` |

---

## 1 一条命令做什么

| 步骤 | 动作 | 只在哪个范围执行 |
|---|---|---|
| ① 同步 docs | 从 `前端交付包/` 复制 5 个文件 | `docs` / `all` |
| ② 同步方法论 | 按 `METHOD_DIRS` 复制 `.md`，**逐篇脱敏**，删除已下线文档 | `method` / `all` |
| ③ 修 README 地址 | 从 `git remote` 推导 Pages 地址并回写 | 总是 |
| ④ 闸门 | 见 §5 | 按范围裁剪 |
| ⑤ 提交 | `git add -A` → 无变化则跳过 → `commit -F <UTF-8 临时文件>` | 总是 |
| ⑥ 推送 | `push origin HEAD:main`；`--tag` 时补推标签 | 总是 |

## 2 参数

| 参数 | 作用 | 默认 |
|---|---|---|
| `--scope docs` | **只发前端展示**（月度更新用这个） | ✅ 默认值 |
| `--scope method` | 只发方法论（逻辑改了才用） | — |
| `--scope all` | 两者都发 | — |
| `--dry-run` | 只同步与检查，不提交不推送 | 关 |
| `--no-push` | 本地提交，不推 | 关 |
| `--tag` | 打 `v5.42.95` 形式标签并推送（`scope=method` 时自动忽略，因为展示没变） | 关 |
| `--message "..."` | 自定义提交信息 | 按范围自动生成 |
| `--repo <路径>` | 换发布仓库目录 | `<发布目录>/qidu` |

提交信息自动生成规则：

| 范围 | 提交信息 |
|---|---|
| `docs` | `data: 月度榜单更新 v5.42-95` |
| `method` | `docs: 方法论更新` |
| `all` | `release: 同步榜单与方法论 v5.42-95` |

退出码：**0 = 成功或无变化；2 = 闸门未通过**。

## 3 为什么这样分（设计理由）

1. **提交历史要能读**：每月一条 `data: 月度榜单更新`，逻辑改动一条 `docs: 方法论更新`，回头看就知道哪次动了口径。
2. **标签只标数据版本**：`v5.42.95` 的含义是"这一版榜单"，不是"这一版文档"，所以 `scope=method` 不打标签。
3. **减少误伤**：方法论有 20 份文件、每份都要过脱敏与路径闸，月更时反复重写容易把偶然改动一起带出去。
4. **回滚清晰**：展示出问题只回滚一条 `data:` 提交，不会连方法论一起退。

## 4 输入契约

| 输入 | 路径 | 缺了怎么办 |
|---|---|---|
| 前端交付包 | `<项目根>/前端交付包/` 5 文件 | 报错退出，不猜 |
| 各 skill 文档 | `<项目根>/skills/<编号>/SKILL.md` 等 | 目录不存在则跳过 |
| 主榜 | `<项目根>/MultiBrand_Ranking_Data.json` | 读不到版本号只跳过版本闸 |
| 发布仓库 | `<发布目录>/qidu`（`--repo` 可改） | 不是 git 仓库直接退出 |

## 5 闸门（按范围裁剪）

| 闸 | 判据 | 生效范围 |
|---|---|---|
| docs 数量 | `docs/` 必须正好 5 个文件 | `docs` / `all` |
| 版本同源 | `docs/data.js` 版本 = 主榜版本 | `docs` / `all` |
| 红线文件 | 仓库内无 `.py` / `cookie` / `profile` / `token` / `config.json` / `registry.json` / `.bak` / `.log` / `.sqlite` | 全部 |
| 路径残留 | md 与 README 内无本机盘符路径、无内部项目名、无用户目录 | 全部 |

## 6 输出契约

| 产物 | 位置 |
|---|---|
| 终端摘要（范围、同步数、闸门、结论） | stdout |
| 发布记录 | `<项目根>/reports/_skill14_<ts>.json`（含 `scope`、变更清单、commit、tag） |
| 远端 | `main` 新提交；`--tag` 且非 method 时附标签 |

## 7 2026-11-23 首发的六个坑（已内建规避）

| 坑 | 现象 | 现在怎么避免 |
|---|---|---|
| 两条命令粘成一行 | `文件名、目录名或卷标语法不正确` | 一条命令一次执行，脚本内部全用 subprocess |
| 账号密码当 token | `Password authentication is not supported` | 走 SSH 密钥，不再依赖 token |
| remote 用户名写错 | `Authentication failed for '.../cory/qidu.git'` | ③ 从 remote 现取现用，不手抄 |
| 仓库名与 README 不一致 | 页面 404 | ③ 自动回写 Pages 地址 |
| 中文提交信息乱码 | 线上显示 `鍦ㄧ嚎鍦板潃` | ⑤ 用 UTF-8 临时文件 + `commit -F` |
| 打标签缺身份 | `Committer identity unknown` | 首次发布配 `git config user.name/email`；脚本只读不改 |

## 8 与其它 skill 的关系

- **13_pre_publish_gate** 管"数据能不能发"（环境 + 主榜校验 + 交付包同源）；**14** 管"发出去"。
- 月度顺序：`07 改主榜 → export_datajs → build_tiers_v3 → 13 → 14 --tag`。
- 逻辑改动顺序：改 `SKILL.md` / 判读手册 → `14 --scope method`（不用重跑 13）。

## 9 首次发布（仓库还没建）

见 `<发布目录>/发布指引_20261123.md`：注册 → 建空仓库（四项全不勾）→ 生成 SSH 密钥并登记 → 初始化本地仓库。完成一次后，以后都用本 skill。

## 10 命令

```bash
# 月度：只发榜单展示（默认范围，可省略 --scope docs）
python skills/14_publish_github/run.py --tag

# 逻辑改动：只发方法论
python skills/14_publish_github/run.py --scope method

# 先看会改什么
python skills/14_publish_github/run.py --dry-run
```
