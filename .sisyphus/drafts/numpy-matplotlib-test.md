# Draft: NumPy + Matplotlib 简单测试代码

## Requirements (confirmed)
- 用户希望得到一个使用 `numpy` 与 `matplotlib.pyplot` 的简单测试代码

## Technical Decisions
- 先采用最小可运行示例（生成 x/y 并画线图）

## Research Findings
- 当前无需额外外部研究，属于简单请求

## Open Questions
- 期望的图类型未确认（折线图/散点图）
- 是否需要保存图片到文件未确认

## New Issue (PyCharm Runtime Error)
- 用户运行 `plt.show()` 时在 PyCharm backend 报错：
  - `AttributeError: 'FigureCanvasInterAgg' object has no attribute 'tostring_rgb'`
- 错误位置指向：`pycharm_matplotlib_backend/backend_interagg.py`

## Preliminary Diagnosis
- 高概率为 **PyCharm 内置 matplotlib backend 与当前 matplotlib 版本兼容性问题**，而非用户绘图代码本身错误。

## Candidate Fixes
- 方案A（推荐，最快）：关闭 PyCharm 的 SciView/自定义后端，改用系统窗口显示。
- 方案B：在代码里强制切换后端为 `TkAgg`。
- 方案C：调整 matplotlib 版本（降级到与当前 PyCharm 兼容版本）。

## Scope Boundaries
- INCLUDE: 提供最小测试示例方向
- EXCLUDE: 复杂训练流程、项目级封装
