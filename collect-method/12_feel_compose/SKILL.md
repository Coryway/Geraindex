# skill 12 · 手感合成（四轴 → 0–100 手感分）

> 定位：把 skill 08 产出的四个轴，合成为**一个可比较的 0–100 手感分**，并落盘到主榜。
> 分工：08 负责「规格 → 四轴」，12 负责「四轴 → 手感分」。**公式只在本文件对应的脚本里实现一次。**

---

## 1 为什么单独抽一个 skill

2026-11-16 之前，`scores.feel` 是在一次性 PowerShell 脚本里算的（`_fix_keipro_axis_20261116.ps1`），
换个环境就无法复现。四轴画像能重算、手感分却算不出来，等于榜单只有一半可重现。

现在：`compose_feel.py` 是公式的唯一实现，08 通过 `from compose_feel import compose_axes` 调用它。

---

## 2 公式（已写入主榜 `meta.feelCompose`）

```
feel = 0.7856932151 × mean(可用轴) + 8.0942272820
```

- 四轴等权
- **缺失轴不参与平均**（不补 50、不补均值）——`n_axes` 记录实际参与轴数
- 结果保留 4 位小数，与主榜存储精度一致
- 常数来源：由 v5.37 主榜 95 款反解（`feel = a×mean + b`，残差 < 1e-10）

---

## 3 用法

```bash
PY="python"

# 校验：profile 复算 vs 主榜 95 款，容差 5e-4
"$PY" skills/12_feel_compose/compose_feel.py --verify

# 导出：{key: {axes, mean, n_axes, feel, resolution}}
"$PY" skills/12_feel_compose/compose_feel.py --emit out.json

# 落盘：写 scores.feel + feelAxes，并升版 vX.YZ（--no-version-bump 可跳过）
"$PY" skills/12_feel_compose/compose_feel.py --apply
```

`--verify` 不通过（任一款差异 > 5e-4 或 profile 缺款）时，`--apply` **直接拒绝执行**，不写任何文件。
`--apply` 前自动备份主榜到 `_bak_code_<YYYYMMDD>/`。

---

## 4 落盘产出

### 4.1 `scores.feel`
仅在差异 > 5e-4 时重写；正常情况下不改动任何已有值（避免舍入造成的无意义 churn）。

### 4.2 `items[].feelAxes`（派生态字段，v5.39 起）

```json
"feelAxes": {
  "bend_load": 92.4479, "neck_fill": 28.5981,
  "board_curve": 99.4755, "press_depth": 100.0,
  "mean": 80.1304, "n_axes": 4,
  "resolutions": {"bend_load":"measured","neck_fill":"measured",
                  "board_curve":"measured","press_depth":"enum_ordinal"}
}
```

前端可直接读取，不需要自己算；`resolutions` 用于标注「实测 / 插补 / 序数」，
配置表里插补的握持轴应显示为估计值（带「估」标）。

---

## 5 验收线

| 检查 | 门槛 |
|---|---|
| profile 覆盖 | 95/95 正式款 |
| 复算与主榜差异 | 最大 ≤ 5e-4（当前 1e-4） |
| 名次 / M / O / T | **零变化**（本 skill 只碰 feel 与其派生态） |
| 非正式款 | 不写 feelAxes（观察/出榜区不参与合成） |

---

## 6 变更记录

- **v1.0（2026-11-16）**：从一次性脚本抽出；新增 `feelAxes` 派生态；主榜 v5.38 → v5.39。
  95 款 feel 逐款复算 0 差异，名次未动。
