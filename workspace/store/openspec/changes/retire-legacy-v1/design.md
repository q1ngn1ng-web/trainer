# retire-legacy-v1 · 设计

## D0 删除清单（2026-11-23 静态导入分析，基线 b825f33）

判定方法：解析 `src/**/*.py` 的 import，从 `src.api.app` 与 `src.cli.__main__` 出发求可达集。
可达 = 现役；只从 `src.main`（Streamlit 入口）可达 = 一代界面；任何入口都不可达 = 死代码。
复现脚本思路：`ast` 收集 `src.*` 的 import 后做图遍历，**只解析不执行**。

### 一代界面专属（只能从 `src/main.py` 到达，3,444 行）

| 路径 | 行数 |
| --- | --- |
| `src/main.py` | 168 |
| `src/ui/page_daily.py`、`page_new_training.py`、`page_training.py`、`page_home.py`、`page_path.py`、`page_sources.py`、`page_review.py`、`theme.py`、`components.py` | 2,661 |
| `src/services/daily_log_service.py` | 224 |
| `src/services/schedule_service.py` | 205 |
| `src/services/recall_service.py` | 134 |
| `src/core/content_dim.py` | 52 |

外加 `src/ui/codemap.md`、`.streamlit/config.toml`。

### 一代生成流程与死代码（任何入口都不可达，1,448 行）

| 路径 | 行数 | 原职责 |
| --- | --- | --- |
| `src/services/trainer_service.py` | 652 | 一代主流程 |
| `src/services/question_service.py` | 174 | 无人引用；出题已由 `answer_service.ensure_question` 承担 |
| `src/services/scoring_service.py` | 164 | 基线评分 |
| `src/services/baseline_service.py` | 151 | 基线出题 |
| `src/services/file_writer.py` | 79 | 把生成的 markdown 落盘 |
| `src/services/renderer.py` | 61 | 一代模板渲染（模板已在 89c2909 删除） |
| `src/services/keyword_service.py` | 60 | 关键词白名单 |
| `src/core/training.py` | 55 | 一代训练对象 |
| `src/services/topic_validation.py` | 52 | 主题具体性校验 |

### 不要删（容易误伤）

- `src/services/calibration_service.py`：周复盘在用（`review_service` 调用）。
- `src/core/baseline.py`、`src/core/element.py`：被 `src/db/models.py` 引用。
- `src/db/connection.py`：现役可达。
- `workspace/store/openspec/archive/` 里的一代 change：是历史记录，保留。

## D1 提示词与模型层收缩

删除 6 个无调用方的用途：`topic_validation`、`keyword_generation`、`baseline_q`、
`baseline_scoring`、`md_generation`、`pretrain_checklist`。

| 文件 | 动作 |
| --- | --- |
| `src/llm/prompts.py` | 删除上述 6 个模板与版本常量，`PROMPT_REGISTRY` 收缩为 9 项 |
| `src/llm/schema.py` | 删除对应的 6 个 Schema 与注册项 |
| `src/llm/fallback.py` | 删除只服务这些用途的静态降级（`baseline_q` 等）；**保留** `goal_clarification` 的表单式降级 |
| `src/llm/validators.py` | 删除 `keyword_coverage_check` |
| `src/llm/client.py` | `KEYWORD_COVERAGE_PURPOSES` 清空、`_semantic_check` 与 `_coerce_keywords` 随之删除；`LIGHT_PURPOSES` 清空为 `frozenset()`，`model_for` 与 `LLM_LIGHT_MODEL` 保留 |

保留的 9 个用途：`goal_clarification`、`edge_probe`、`edge_probe_grade`、`path_skeleton`、
`quiz_variant`、`quiz_grade`、`explanation`、`train_question`、`weekly_calibration`。

## D2 测试处置

| 文件 | 动作 |
| --- | --- |
| `tests/test_plan_page.py`、`tests/test_source_page.py` | 删除（Streamlit 页面测试） |
| `tests/test_question_service.py` | 删除（被测模块无人引用） |
| `tests/test_units.py` | 拆分：删除一代渲染器、`trainer_service`、`daily_log`、`recall`、`schedule` 相关用例类；**保留**排期、掌握度等现役纯函数用例 |
| `tests/test_model_routing.py` | 改写：配置了 `LLM_LIGHT_MODEL` 时，9 个现役用途全部仍走主力模型 |
| `tests/test_prompt_schema_contract.py` | 改写：版本钉收缩到 9 个现役用途，「版本不得回退」守卫继续保留 |
| `tests/test_llm_audit.py` | 改写：用途冒烟覆盖收缩到 9 个 |

实施时以 `grep -nE "streamlit|trainer_service|renderer|recall_service|schedule_service|daily_log_service|question_service|topic_validation|keyword_service|baseline_service|scoring_service|file_writer|content_dim|core\.training" tests/*.py` 的结果为准。

## D3 脚本处置

| 脚本 | 动作 |
| --- | --- |
| `scripts/e2e_full_flow.py`、`scripts/seed_test_data.py`、`scripts/harness_smoke.py` | 删除（跑的都是一代流程） |
| `scripts/audit_coverage_smoke.py`、`scripts/check_llm_logs.py` | 改写：用途清单去掉一代用途 |
| `scripts/eval_purpose_model_routing.py` | 删除（它评测的轻语义用途正是一代用途；路由机制由单测覆盖） |
| `scripts/backfill_question_meta.py`、`benchmark_time_cost.py`、`eval_grading_consistency.py`、`eval_grading_memory_cost.py`、`eval_schema_first_pass.py`、`explain_grading.py` | 保留，逐个确认 import 仍可解析 |
| 新增 `scripts/e2e_api_smoke.py` | 不需要 API key：用桩替代模型调用，走通 `wizard/start → pace → confirm → sources → probe → activate → today → question → answer`，每步断言状态码与关键字段 |

新增的冒烟脚本填补一个空白：`oss-release-gate` 的 CI 需要一条不花钱的端到端门禁。

## D4 依赖与配置

- `requirements.txt`：删除 `streamlit`、`jinja2`（后者只被 `renderer` 使用）。
- 删除 `.streamlit/config.toml`。
- `.env.example`：删除 `TRAINING_ROOT`（一代落盘目录）与 `STREAMLIT_PORT`。
  这两个变量在 `src/`、`scripts/` 里已无读取点（2026-11-23 grep 为零），删除只是清理文档。

## D5 数据层为什么不动

`daily_logs`、`daily_log_tasks`、`baseline_history` 三张表，以及 `trainings` 上的
`directory`、`keywords`、`must_cover_count`、`forbidden`、`targets`、`review_items`、
`pretrain_checklist`、`schedule`、`materials`、`current_week`、`last_review_at` 等列，
在代码删除后不再有读写方。本 change 不删它们：

- 删表删列不可逆，且与「让仓库不再误导贡献者」这个目标无关；
- `src/db/models.py` 的 `Training` 与 `queries.py` 的字段白名单是「接线缺口」高发区
  （模型缺列曾导致权限静默失效），不值得为清理而碰；
- 是否清理由作者看过数据后另行决定。

## D6 验收三道关

1. `grep -rnE "streamlit" src tests scripts requirements.txt` → 零命中。
2. `python -c "from src.llm.prompts import PROMPT_REGISTRY; print(sorted(PROMPT_REGISTRY))"`
   → 恰好是 D1 列出的 9 个。
3. 全量 `pytest tests/ -q` → 0 失败；`cd web && npm run build` → 成功；
   用现有 `data/trainer.db` 启动服务并请求 `/api/health` → 200（证明没有迁移、没有破坏存量库）。

## D7 与既有 change、并行会话的关系

- **`understanding-edge-assessment`（未归档，12/30）对 `baseline-diagnosis` 规格也有一份增量。**
  本 change 会把该规格的全部要求移除。归档顺序：先处理旧 change（归档或关闭其对
  `baseline-diagnosis` 的任务），再归档本 change；若旧 change 长期不归档，在本 change
  归档时手动合并，以本 change 为准。
- **`oss-release-gate`**：本 change 接在它后面做。它改了 README、AGENTS.md、
  `requirements.txt`（补 fastapi / uvicorn 等）；本 change 在同一批文件上继续改
  （去掉 streamlit / jinja2 与一代目录说明）。顺序反过来会把两边的修改互相覆盖。
- 89c2909 已经删除了 `src/templates/*.j2`，但 `renderer.py` 与对应用例还在——
  这正是全量测试里那个常驻失败的来源，本 change 自然收掉。

**冲突避让清单**：动手前 `git status --short`，若并行会话仍有未提交改动，
逐个确认不在本 change 的删除清单里再开始。热区文件：`requirements.txt`、`README.md`、
`AGENTS.md`、`docs/architecture.md`——编辑前重读最新内容。

**改动纪律**：删除用 `git rm`，不要用文件系统删除后忘记暂存；提交只 `git add` 自己的文件。

**禁止触碰项**：`data/trainer.db`；判分、排期、出题、解析的任何现役逻辑；
`src/db/tables.sql` 与 `src/db/migrate.py`；并行会话的未提交改动。
