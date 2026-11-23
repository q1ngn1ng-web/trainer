# 20261123 · chore · 登记 retire-legacy-v1——移除一代遗留代码（仅提案，待实施）

## 变更摘要

- 新增 openspec change **`retire-legacy-v1`**（proposal / design / tasks / 三份 spec 增量，
  `openspec validate --strict` 通过，0/14 任务）。来源：作者 2026-11-23 拍板
  "一代遗留代码都删，另立 change"。本会话不写代码，实施移交另一会话。
- 同批把 `oss-release-gate` 的作者决策回填完毕（0.1 MIT、0.2 全新历史公开仓、
  0.3 视频不放演示地址、0.4 一代代码全删、0.5 开发记录带进公开仓）。

## 边界是怎么定的

对 `src/**/*.py` 做静态导入分析（`ast` 解析 import，只解析不执行）：从 `src.api.app` 与
`src.cli.__main__` 出发求可达集。

| 类别 | 行数 | 内容 |
| --- | --- | --- |
| 现役（API / CLI 可达） | 19,168 | 44 个模块 |
| 只有一代 Streamlit 界面可达 | 3,444 | `src/main.py`、`src/ui/*`、`daily_log_service`、`schedule_service`、`recall_service`、`core/content_dim` |
| 任何入口都不可达 | 1,448 | `trainer_service`、`question_service`、`scoring_service`、`baseline_service`、`file_writer`、`renderer`、`keyword_service`、`core/training`、`topic_validation` |

遗留合计 **4,892 行**，占全量 Python（24,062 行）的 20%。此前的口头估计 4,043 行漏掉了
后两类里的三个 service 与两个 core 模块。

## 提案要点

- 整份作废两份一代规格：`baseline-diagnosis`（4 条要求）、`trainer-generation`（5 条要求）。
- 模型层收缩：15 个提示词删到 9 个；`LIGHT_PURPOSES` 清空但路由机制保留。
- 测试：删 3 个文件、拆 `test_units.py`、改写 3 个。全量测试里那个常驻失败的 renderer 用例
  随之消失。
- 脚本：删 4 个、改写 2 个；新增一个**不需要 API key** 的接口级冒烟脚本，供 CI 使用。
- 数据层完全不动：不删表、不删列、不写迁移，也不碰 `models.py` / `queries.py` 的字段映射
  （"接线缺口"高发区）。

## 依赖与顺序

- 接在 `oss-release-gate` 之后做（两者都改 README、AGENTS.md、requirements.txt）。
- `understanding-edge-assessment`（未归档）对 `baseline-diagnosis` 也有一份增量；
  归档顺序见 change 的 design D7。

## 纪律备注

- 本会话零业务代码改动；只新增 change 四件套与本记录。
- 未触碰并行会话的文件；提交使用 pathspec。
