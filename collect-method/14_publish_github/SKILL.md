# skill 14 · 一键发布到 GitHub

> 版本 v1.0 · 2026-11-23
> 入口：`python skills/14_publish_github/run.py`
> 依赖：git（`<工具路径> 或 `GIT_EXE` 环境变量）、已初始化的发布仓库（默认 `<发布目录>
> **不改主榜、不改前端交付包**；只写发布仓库与 git 记录。

---

## 人话版（先读这段）

以前更新一次榜单要手工做六件事：复制 5 个前端文件 → 复制 18 份文档 → 脱敏 → 检查有没有夹带 `.py` → commit → push。每一步都可能漏，2026-11-23 第一次发布时就在“把两条命令粘成一行”“token 当成密码”“仓库名写错”上耗了半小时。

这个 skill 把六步合成一条命令。它只做两件事：**把最新的东西同步过去**，**在推之前拦住不该发的东西**。

判断很简单：跑完看到 `结论：已推送` 就算完；看到 `FAIL` 就别推，按提示修。

---

## 1 一条命令做什么

| 步骤 | 动作 | 拦什么 |
|---|---|---|
| ① 同步 docs | 从 `前端交付包/` 复制 5 个文件（`index.html` `styles.css` `app.js` `data.js` `tiers.js`） | 缺文件直接报错 |
| ② 同步方法论 | 按 `METHOD_DIRS` 复制各 skill 的 `.md`，**逐篇脱敏**，删除已下线的旧文档 | 路径泄漏 |
| ③ 修 README 地址 | 从 `git remote` 推导 `https://<user>.github.io/<repo>/` 并回写 | 地址与仓库不一致 |
| ④ 闸门 | docs=5 个 / `docs/data.js` 版本 = 主榜版本 / 无 `.py`·凭证·备份 / md 内本机路径 0 命中 | 发错东西 |
| ⑤ 提交 | `git add -A` → 无变化则跳过 → `commit -F <UTF-8 临时文件>` | 空提交 |
| ⑥ 推送 | `push origin HEAD:main`，`--tag` 时补打 `v<版本>` 并推 | — |

## 2 参数

| 参数 | 作用 | 什么时候用 |
|---|---|---|
| 无 | 同步 + 闸门 + 提交 + 推送 | 月更后的标准操作 |
| `--dry-run` | 只同步与检查，不提交不推送 | 想先看会改什么 |
| `--no-push` | 本地提交，不推 | 网络不稳/想先本地看 diff |
| `--tag` | 顺带打 `v5.42.95` 这类标签并推送 | 正式发版 |
| `--message "..."` | 自定义提交信息 | 特殊说明，如修错别字 |
| `--repo <路径>` | 换发布仓库目录 | 仓库搬家 |

退出码：**0 = 成功或无变化；2 = 闸门未通过**。

## 3 输入契约

| 输入 | 路径 | 缺了怎么办 |
|---|---|---|
| 前端交付包 | `<项目根>/前端交付包/` 5 文件 | 报错退出，不猜 |
| 各 skill 文档 | `<项目根>/skills/<编号>/SKILL.md` 等 | 目录不存在则跳过该目录 |
| 主榜 | `<项目根>/MultiBrand_Ranking_Data.json` | 读不到版本号时只跳过版本闸，其余照跑 |
| 发布仓库 | `<发布目录> 可改） | 不是 git 仓库直接退出 |

## 4 输出契约

| 产物 | 位置 |
|---|---|
| 终端六步摘要 + 结论 | stdout |
| 发布记录 | `<项目根>/reports/_skill14_<ts>.json`（含变更清单、状态、Pages 地址） |
| 远端 | `main` 分支新提交；`--tag` 时附标签 |

## 5 三道红线（脚本强制）

1. **`docs/` 必须 5 个文件**，多一个少一个都停。
2. **`docs/data.js` 版本必须等于主榜版本**。防止“改了主榜没重导 data.js”就把旧数据发上线。
3. **发布仓库里不许出现** `.py` / `cookie` / `profile` / `token` / `config.json` / `registry.json` / `.bak` / `.log` / `.sqlite`，以及 md 里的本机绝对路径。

## 6 2026-11-23 首发的六个坑（已内建规避）

| 坑 | 现象 | 现在怎么避免 |
|---|---|---|
| 两条命令粘成一行 | `文件名、目录名或卷标语法不正确` | 一条命令一次执行，脚本内部全用 subprocess |
| 账号密码当 token | `Password authentication is not supported` | 走 SSH 密钥，不再依赖 token |
| remote 用户名写错 | `Authentication failed for '.../cory/qidu.git'` | ③ 从 remote 现取现用，不手抄 |
| 仓库名与 README 不一致 | 页面 404 | ③ 自动回写 Pages 地址 |
| 中文提交信息乱码 | 线上显示 `鍦ㄧ嚎鍦板潃` | ⑤ 用 UTF-8 临时文件 + `commit -F` |
| 打标签缺身份 | `Committer identity unknown` | 首次发布时 `git config user.name/email`，脚本只读不改 |

## 7 与其它 skill 的关系

- **13_pre_publish_gate** 管“数据能不能发”（环境 + 主榜校验 + 交付包同源），**14** 管“发出去的动作”。
- 顺序：`07 改主榜 → export_datajs → build_tiers_v3 → 13 闸门 → 14 发布`。
- 14 不判断榜单对不对，只保证“发出去的就是当前这份”。

## 8 首次发布（仓库还没建）

见 `<发布目录> → 建空仓库 → 生成 SSH 密钥并登记 → 初始化本地仓库。完成一次后，以后都用本 skill。

## 9 命令

```bash
python skills/14_publish_github/run.py --dry-run   # 先看会改什么
python skills/14_publish_github/run.py --tag       # 正式发布并打标签
```
