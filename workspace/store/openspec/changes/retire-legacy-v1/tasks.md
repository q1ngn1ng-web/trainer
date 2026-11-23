# retire-legacy-v1 · 任务清单（未排期）

> 前置：`oss-release-gate` 的 2.1 / 2.3 / 2.4（依赖、README、AGENTS.md）先落地。
> 开工前先读 design D7 的冲突避让清单与「不要删」清单（design D0）。

## 1. 删除代码

- [ ] 1.1 删除一代界面：`src/main.py`、`src/ui/`（含 `codemap.md`）、`.streamlit/`。
- [ ] 1.2 删除只被一代界面使用的服务：`src/services/daily_log_service.py`、
      `recall_service.py`、`schedule_service.py`、`src/core/content_dim.py`。
      删前对每个模块跑一次反向 grep，确认除 `src/ui/`、`src/main.py`、测试与脚本外无引用。
- [ ] 1.3 删除一代生成流程与死代码：`trainer_service.py`、`baseline_service.py`、
      `scoring_service.py`、`keyword_service.py`、`topic_validation.py`、`renderer.py`、
      `file_writer.py`、`src/core/training.py`、`question_service.py`。
      删 `question_service.py` 前先与 `answer_service.ensure_question` 对照，
      确认没有被遗漏的能力；有则在 record 里写明并留给作者决定。
- [ ] 1.4 收缩模型层（design D1）：`prompts.py`、`schema.py`、`fallback.py`、
      `validators.py`、`client.py`。保留 `goal_clarification` 的表单式降级与
      `model_for` 路由机制。

## 2. 测试与脚本

- [ ] 2.1 按 design D2 处置测试文件；`test_units.py` 只删一代用例类，现役用例一行不改。
- [ ] 2.2 按 design D3 处置脚本：删除 4 个、改写 2 个。
- [ ] 2.3 新增 `scripts/e2e_api_smoke.py`：不需要 API key，用桩替代模型调用，走通
      `wizard/start → pace → confirm → sources → probe → probe/grade → activate → today →
      question → answer → session-summary`，每步断言状态码与关键字段。
      它同时是 `oss-release-gate` 任务 3.1（CI）要用的端到端门禁。
- [ ] 2.4 用新的冒烟脚本复现并确认 `oss-release-gate` 任务 1.1 的会话小结 500
      （带 `plan_id` 的作答）：若该任务尚未修复，本脚本应当红；修复后转绿。

## 3. 依赖与文档

- [ ] 3.1 `requirements.txt` 去掉 `streamlit`、`jinja2`；删除 `.streamlit/config.toml`。
- [ ] 3.2 `.env.example` 去掉 `TRAINING_ROOT`、`STREAMLIT_PORT`。
- [ ] 3.3 文档同步：`AGENTS.md`（去掉「`src/` 是 Streamlit demo」「不要重构 demo 代码」、
      启动命令、目录表里的 `src/ui`）、根目录与 `src/` 的 `codemap.md`、
      `docs/architecture.md` 中的一代描述、README 目录结构里的「已下线的一代界面」一行
      （删除后该行不再需要）。

## 4. 验收与归档

- [ ] 4.1 design D6 的三道验收关全部通过，数字写进 record（删除行数、测试通过数）。
- [ ] 4.2 全新环境按 README 复测一次（`uv venv` → 安装 → `init-db` → 前端构建 → 启动），
      确认安装体积与耗时较删除前下降。
- [ ] 4.3 record（类型 `refactor`）+ 归档；归档顺序见 design D7。
