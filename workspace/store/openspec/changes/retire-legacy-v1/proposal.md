# retire-legacy-v1 · 移除一代遗留代码（提案，待实施）

> 状态：**仅提案，未实施**。本文件为**自包含交接文档**——实施会话只读本 change
> （proposal / design / tasks / specs）即可动手，不需要创建者的对话上下文。
> 来源：作者 2026-11-23 拍板「一代遗留代码都删，另立 change」（`oss-release-gate` 的决策 0.4）。

## Why

Trainer 有两代产品。一代是 Streamlit 界面加「十要素文件生成」流程（主题校验 → 关键词 →
基线诊断 → 生成 10 份 markdown → 每日任务卡）。二代是现役的 FastAPI + Vue 向导流程
（澄清目标 → 导入资料 → 理解边缘探测 → 路径骨架 → 排期）。二代已完整取代一代，
但一代的代码还留在仓库里，带来四个问题：

1. **误导贡献者**：README 的 Quick Start 与「测试」一节长期指向一代入口
   `streamlit run src/main.py`（已由 `oss-release-gate` 更正），AGENTS.md 仍称
   `src/` 是「仅作参考的 Streamlit demo」。
2. **测试里混着一代用例**：全量测试里唯一常驻的失败项
   `test_units.py::TestRendererEmpty::test_render_empty_context_returns_ten_files`
   就来自一代渲染器——模板已在 89c2909 被删除，渲染器与用例还在。
3. **拖慢安装**：`streamlit` 是一代界面的专属依赖，现役产品完全用不到。
4. **死提示词占着注册表**：15 个注册提示词里有 6 个没有任何现役调用方，
   审计与评测脚本仍在为它们兜底。

一代代码在现役产品里**不可达**：从 `src/api/app.py` 与 `src/cli` 出发做静态导入分析，
下面这些模块没有任何路径能到达（清单与行数见 design D0）。

## What Changes

**A. 删除一代界面**：`src/main.py`、`src/ui/` 全部、`.streamlit/`，以及只被它们使用的
`daily_log_service` / `recall_service` / `schedule_service` / `core/content_dim`。

**B. 删除一代生成流程**：`trainer_service` / `baseline_service` / `scoring_service` /
`keyword_service` / `topic_validation` / `renderer` / `file_writer` / `core/training`，
以及无人引用的 `question_service`。

**C. 收缩模型层**：删除 6 个无调用方的提示词及其 Schema、静态降级与关键词覆盖率校验；
`LIGHT_PURPOSES` 清空但路由机制保留。

**D. 清理测试与脚本**：删除一代用例；脚本删除 4 个、改写 2 个，
新增一个不需要 API key 的接口级冒烟脚本（替代一代的 `e2e_full_flow.py`）。

**E. 依赖与文档**：`requirements.txt` 去掉 streamlit 与 jinja2；`.env.example` 去掉
两个一代专用变量；AGENTS.md、codemap、README、架构文档同步。

### 关键取舍

| 选择 | 备选 | 放弃理由 |
| --- | --- | --- |
| 直接删除 | 移入 `legacy/` 目录 | 作者拍板删除；打标签即可回溯，留目录只会继续误导 |
| 数据层完全不动 | 顺带删除一代专用的表与列 | 删表不可逆且与代码解耦无关；列的清理由作者另行决定 |
| 保留模型路由机制 | 连 `model_for` 一起删 | 路由是按用途的通用能力，只是暂时没有轻语义用途 |
| 判定依据用静态导入可达性 | 按文件名与文档描述判断 | 文档本身已失实；导入图可复现 |

## 非目标（Non-goals）

- 不改任何现役流程的行为：判分、排期、出题、解析、分享、权限一律不动
- 不删表、不删列、不写迁移（`daily_logs`、`daily_log_tasks`、`baseline_history` 与
  `trainings` 上的一代专用列保持原样）
- 不改 `src/db/models.py` 与 `queries.py` 里的一代字段映射（见 design D5）
- 不做 agent 化、不换底座

## Capabilities

### Modified Capabilities

- `baseline-diagnosis`：整份规格作废（一代基线诊断已不在现役流程里）
- `trainer-generation`：整份规格作废（一代十要素文件生成已不在现役流程里）
- `llm-observability`：移除关键词覆盖率校验；新增「提示词注册表只含有调用方的用途」

## Impact

- **代码**：删除约 4,892 行（清单见 design D0）；`src/llm/` 四个文件小改
- **测试**：删除 3 个测试文件与 `test_units.py` 中的一代用例，改写 3 个；预期全量
  0 失败（现役唯一常驻失败项随之消失）
- **脚本**：删除 4 个、改写 2 个、新增 1 个
- **数据**：零迁移
- **风险**：中——删除面大，但全部是不可达代码；主要风险是删到仍被引用的模块，
  用 design D6 的三道验收命令兜底
