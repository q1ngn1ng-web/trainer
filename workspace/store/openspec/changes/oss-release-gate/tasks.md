# oss-release-gate · 任务清单（部分已实施）

> 分四段，可按需截断移交：**1.x 录制前必须绿；2.x 公开仓库前必须绿；
> 3.x 一周内；4.x 条件触发**。第 0 节是作者决策。开工前先读 design D9 的冲突避让清单。
> 标 `[x]` 的是规划会话在 2026-11-23 直接完成的文档类任务，均已在全新环境实测。

## 0. 作者决策

- [x] 0.1 许可证：**MIT**（2026-11-23 拍板；`LICENSE` 已落盘，见 2.6）。
- [x] 0.2 公开仓策略：**全新历史公开仓**（2026-11-23 拍板）。落地方式：旧的
      `personal_trainer_agent` **保持公开不动**，新公开仓另起名字，从白名单导出后推送。
      旧仓自 2026-09-14 起就是 public，历史里一直带着面试材料与开发记录——这一点作者知情。
- [x] 0.3 视频里**不放**演示地址（2026-11-23 拍板）——4.x 维持"条件触发"，不阻断发布。
- [x] 0.4 一代遗留代码**全部删除，另立 change** `retire-legacy-v1`（2026-11-23 拍板）。
      边界按静态导入分析是 **4,892 行**，比最初估的 4,043 行多出三个 Streamlit 专属 service、
      `core/content_dim`、`core/training` 与无人引用的 `question_service`。
- [x] 0.5 个人开发记录**带进公开仓**（2026-11-23 拍板）：`workspace/record/`、
      `BACKLOG.md`、`HANDOFF.md` 公开；面试材料、含真实作答的标注文件、`data/` 下的 PDF、
      个人工作提示词与汇总文档仍不进公开仓。
- [ ] 0.6 **待拍板**：个人免登录模式（`TRAINER_AUTH=0`）去留。实测该模式下新建训练的第一步
      `POST /api/wizard/start` 返回 401——它和另外 17 个端点一样硬性要求登录用户
      （`_require_user` / `require_viewer`），而免登录模式下没有用户。推荐**废弃**：
      注册已开放且只存本地，免登录模式只剩误导；实施见 2.8。
- [ ] 0.7 **待拍板**：开发主仓与公开仓的关系。2026-11-24 已按 0.2 落地：新建公开仓
      `q1ngn1ng-web/trainer`（全新历史，单提交快照），旧仓保持公开不动。但本地开发仓的
      `origin` 仍指向旧仓，新公开仓只是一次性快照，**从现在起两边开始分叉**。推荐把
      `trainer` 变成唯一的开发主仓：把不进公开仓的私人材料移到仓库之外，本地仓改指新仓；
      过渡期至少要有一条"本地 → 公开仓"的同步流程。无论选哪种，都**禁止**把本地完整历史
      直接 push 到 `trainer`——那会毁掉"全新历史"。

## 1. 录制前必须绿

- [ ] 1.1 修会话小结 500：`src/services/session_summary.py::summarize_day` 的元数据查询
      不得引用 `plan_items` 上不存在的列。知识点名按 design D2 的顺序解析
      （`question_bank.knowledge_point` → `training_items.knowledge_point` → `item_key`），
      要点清单仍取 `plan_items.points`；单条元数据缺失不得让整份小结抛异常。
- [ ] 1.2 端点级回归钉子（`tests/test_coach_memory.py`）：补"带 `plan_id` 的真实计划项"
      链路——建训练与计划项 → `record_attempt(plan_id=...)` → 请求
      `GET /api/trainings/{id}/session-summary`，断言 200、题数正确、
      `weakest_knowledge_point` 有值。保留原有空日用例。
- [ ] 1.3 澄清追问入史（`src/services/goal_clarification_service.py`）：`_persist` 把本轮
      待答问句写进 `goal_json`（键 `pending_question`）；`continue_session` 追加 history 时
      用上一轮的 `pending_question` 作为 `question`；旧训练无该键时回落空串（design D3）。
- [ ] 1.4 测试（`tests/test_goal_clarification.py`）：两轮澄清后 `history[0]["question"]`
      等于第 1 轮的追问；经 `load_session` 恢复后再回答，历史仍带问句；无 `pending_question`
      的旧状态不报错。
- [ ] 1.5 衰减时间口径（`src/services/learner_service.py`）：`_parse` 与 `get_learner_model`
      改为按真实流逝时间计算——带时区按其时区、不带时区按 UTC，与
      `datetime.now(timezone.utc)` 相减（design D4）。测试用带 `+00:00` 的时间戳断言
      "刚作答完 → 距 last_seen 约为 0、`decayed_mastery == mastery`"。
- [ ] 1.6 "当天"口径（`src/services/session_summary.py`）：把 `date(created_at, 'localtime')`
      换成按 `local_tz()` 计算的当日 UTC 起止区间过滤（design D4）。测试：把 `TRAINER_TZ`
      设为非系统时区，断言跨日的两条作答落在各自的日子；测试结束恢复原值。
- [ ] 1.7 基线出题提示词错别字（`src/llm/prompts.py:36`）："哟怒火" → "用户"；
      `BASELINE_Q_PROMPT_VERSION` v1.0.0 → v1.0.1，并同步
      `tests/test_prompt_schema_contract.py` 的版本钉。该提示词属一代流程、会被
      `retire-legacy-v1` 整个删掉；若先做那个 change，本项可跳过。
- [ ] 1.8 全量回归：`.venv/bin/python -m pytest tests/ -q -p no:cacheprovider`。
      实施前基线 415 passed / 6 skipped / 1 failed（renderer 模板那条属一代代码，不计入）；
      本 change 不得新增失败。改后 `sudo -n systemctl restart trainer-api`，
      用作者本人的账号对一个**当天练过日常题**的训练请求 `session-summary`，确认 200
      （线上复现用的是训练 144，2026-11-23 17:09 的两次 500）。

## 2. 公开仓库前必须绿

- [x] 2.1 `requirements.txt`：已补 fastapi、uvicorn、pytest 与测试所需的 httpx，
      删除与 python-dotenv 重复的 `dotenv`。`python-multipart` 经核对并无代码依赖，不列入。
      streamlit 与 jinja2 只被一代代码使用，随 `retire-legacy-v1` 移除。
      `rich` 疑似没有任何引用，留给实施会话确认后删除。
- [x] 2.2 `.env.example`：已整理——API key 留空（原值是非空的占位符 `sk-your-api-key-here`，
      复制即用会让每次模型调用真的去请求三次并吃三次 401，而不是立即降级）；
      地址与模型名对齐代码默认值；补 `TRAINER_AUTH`、`TRAINER_COOKIE_SECURE`；
      一代专用变量归到"遗留"段。**没有收录 `TRAINER_TZ`**，原因见 2.9。
- [x] 2.3 README 重写（design D7 为验收标准）：已按现实重写为 FastAPI + Vue 的真实启动方式，
      含快速开始、环境变量表、常见问题、架构、测试、证据边界、路线图、参与贡献、License。
      **还差一张界面截图或 GIF**（README 源码里有位置注释），由作者录制时顺手截。
- [ ] 2.4 AGENTS.md 更正失实描述（只更正、不改规则）：`src/` 不再是"仅作参考的 Streamlit
      demo"，入口是 `src/api/app.py`，启动命令同 README；目录表补 `web/`；
      "不要重构 src/ 下的 demo 代码"一条按 0.4 的决定改写适用范围。该文件是所有 AI
      协作者的入口文档，失实会误导每一个贡献者。
- [x] 2.5 全新环境冒烟：已按 README 实测两轮。要点：`python -m venv` 在这台 Ubuntu 上
      因缺 `python3.12-venv` 直接失败，所以 README 以 uv 为主路径、venv 为折叠备选；
      直连 PyPI 在此网络下超时，加清华镜像后依赖 22 秒装完；npm 换国内镜像后 12 秒；
      **从零到启动共 35 秒**。页面、静态资源、注册、登录、仪表盘均 200。
      详见 record `20261123_docs_readme-oss-release.md`。
- [x] 2.6 LICENSE 落盘（MIT）+ README 的 License 一节。
- [x] 2.7 私人材料出库：全新历史公开仓方案下等于"导出时走白名单"。不进公开仓的有：
      `workspace/面试/`、`workspace/evidence/判分一致性标注*`（含真实作答的四个文件）、
      `data/*.pdf`、`workspace/prompts/`、`workspace/AUDIT-BRIEF.md`、`workspace/SUMMARY-*.md`。
      旧仓保持原样（0.2）。
- [ ] 2.8 个人免登录模式的处置（依赖 0.6）。若废弃：`TRAINER_AUTH=0` 时启动日志给出明确警告并
      按 `1` 处理；`.env.example` 与 README 去掉对该模式的说明。测试：`TRAINER_AUTH=0` 启动后
      `GET /api/auth/mode` 返回 `auth_enabled=true`。
- [ ] 2.9 `TRAINER_TZ` 写进 `.env` 不生效：`src/core/plan.py` 在模块导入时就读了这个环境变量，
      早于 `load_dotenv()`。实测 `.env` 里写 `TRAINER_TZ=UTC` 得到的仍是 `Asia/Shanghai`，
      直接设在环境变量里才生效。修法二选一：把读取挪到 `local_tz()` 调用时，或在 `core/plan.py`
      顶部先 `load_dotenv()`。修后把变量加回 `.env.example`，并删掉 README 表格里的那句提示。
      测试：临时改写 `.env` 后断言 `local_tz()` 随之变化，测试结束恢复。
- [ ] 2.10 `docs/architecture.md` 的"项目定位"段仍是早期口径（"MVP 阶段只有用户本人（自用）"、
      十要素文件生成），与现役的向导流程不符。更新后可从 README 的架构一节链接过去。
- [ ] 2.11 四个评测脚本写死了 `/home/ubuntu/...` 绝对路径（`eval_grading_consistency.py`、
      `eval_grading_memory_cost.py`、`eval_purpose_model_routing.py`、`eval_schema_first_pass.py`），
      外人克隆下来跑不了。改为相对项目根目录解析。其中 `eval_purpose_model_routing.py` 会随
      `retire-legacy-v1` 删除，可以先跳过。

## 3. 一周内（贡献者基础设施）

- [ ] 3.1 CI：`.github/workflows/ci.yml`——后端 pytest（Python 3.12）+ 前端
      `npm ci && npm run build`（Node 22）。一代渲染器那条常驻失败若还在，先跳过该用例，
      待 `retire-legacy-v1` 落地后恢复全量。
- [ ] 3.2 `CONTRIBUTING.md`：本地启动、目录导览、openspec 流程怎么参与、提交信息规范、
      测试要求（可见性 / 权限 / 过滤类改动必须带端点级用例与跨用户隔离用例）。
- [ ] 3.3 `.github/ISSUE_TEMPLATE/`（bug 报告、功能建议、good first issue）与
      `PULL_REQUEST_TEMPLATE.md`。
- [ ] 3.4 `SECURITY.md`：私下报告漏洞的方式；明示演示实例（如有）不保证可用性。
- [ ] 3.5 good first issue 清单（至少 6 个，各配验收标准），建议来源：统一全项目 `_now()`
      时间口径、澄清追问入史的前端展示、出题资料接地的检索化、`app.py` 按业务域拆分路由、
      前端首个组件测试、部署文档。
- [ ] 3.6 `deploy/`：systemd 单元与 nginx 反代样例（路径与域名占位化），README 链接。

## 4. 暴露面护栏（放演示地址则为发布前阻断项）

- [ ] 4.1 注册开关 `TRAINER_REGISTER_OPEN`（默认 1 保持现状；为 0 时
      `POST /api/auth/register` 返回 403 与可读提示，登录不受影响）+ 测试。
- [ ] 4.2 限流：登录、注册、出题、判分、澄清端点按来源 IP 限流（进程内令牌桶，
      阈值走环境变量），超限 429 + `Retry-After` + 测试。
- [ ] 4.3 每用户每日 LLM 调用上限（按 `llm_calls` 计数，阈值走环境变量）：超限时出题、
      判分等入口返回可读错误，不调用 LLM、不写半成品 + 测试。
- [ ] 4.4 通用 500 处理：未捕获异常返回 JSON（含请求追踪 id），服务端日志带同一 id，
      便于观众报 bug 时直接贴。
- [ ] 4.5 部署走 HTTPS：nginx 反代 + 证书，`TRAINER_COOKIE_SECURE=1`，uvicorn 只监听
      127.0.0.1（样例见 3.6）。
- [ ] 4.6 用户偏好按用户隔离（`src/services/preferences_service.py`、`src/api/app.py` 的
      `GET/PUT /api/preferences`、`src/services/answer_service.py:626`）：键带用户标识，
      读取顺序「用户键 → 存量全局键 → 默认值」，写入只写用户键（design D8）。
      测试：双账号互不改写（A 关掉连败提示后 B 的 `submit_answer` 仍返回 `coach_hint`）；
      未带用户标识的存量值仍能被读到。
- [ ] 4.7 关闭接口文档：`/docs`、`/redoc`、`/openapi.json` 不在 `/api` 前缀下，鉴权中间件
      不拦，实测未登录即可读取全部接口定义。`TRAINER_AUTH=1` 时关闭（`FastAPI(docs_url=None,
      redoc_url=None, openapi_url=None)`），`TRAINER_AUTH=0` 的本地模式保留。测试：开启登录时
      三个地址返回 404。
