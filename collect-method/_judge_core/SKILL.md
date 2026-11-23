# _judge_core · 判读通用核心

> 定位：所有"人/模型判读"环节的公共工具。不产出分数，只保证判读**可复现、可检验**。

## 生产用（月度必看）

| 文件 | 作用 |
|---|---|
| `build_judge_pack.py` | 生成盲判包：拾音器型号归一化、品牌/型号/产地/价格打码、评论只取声音句 |
| `agreement.py` | 一致性检验：MAE / bias / r / rho / 档位一致率 / QWK / Bland-Altman |
| `MONTHLY_JUDGE_SOP.md` | **月度判读 6 步 + 验收线**，配合 `09_tone/JUDGE_RUBRIC.md` 使用 |

```bash
PY="python"
"$PY" skills/_judge_core/build_judge_pack.py --keys "A|B,C|D" --out_pack P.json --out_key K.json
"$PY" skills/_judge_core/agreement.py --a 旧.json --b 新.json --bands 56,60,64,68,72,76
```

## 其余为内部模块

`attr.py` / `cache.py` / `classify.py` / `cursor.py` / `key.py` / `label_pool.py` /
`prompt.py` / `schema.py` / `video_gate.py` / `selftest.py` —— 供 05/06/11 的评论编码复用，不单独调用。

## 铁律

判读前必须先出盲判包。**直接看着榜单判，等于把价格和品牌判进去**（实测价格溢价 3.4 分/倍价，见 `_过目/273`）。
