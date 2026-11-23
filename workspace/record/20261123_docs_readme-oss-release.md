# 20261123 · docs · README 按现役产品重写 + MIT 许可证 + 依赖与配置模板对齐现实

## 变更摘要

- `README.md` 全文重写：从"Streamlit 一代 MVP"改为现役的 FastAPI + Vue 产品。含快速开始、
  环境变量表、常见问题、架构、测试、证据边界、路线图、参与贡献、License。
  开头如实写明"仓库名里有 agent，但它现在不是 agent"。
- 新增 `LICENSE`（MIT，作者 2026-11-23 拍板）。
- `requirements.txt`：补 fastapi、uvicorn、pytest、httpx（测试客户端依赖）；删除与
  python-dotenv 重复的 `dotenv`。`python-multipart` 经核对无代码依赖，未列入。
- `.env.example`：API key 由占位符改为留空；地址与模型名对齐代码默认值；补
  `TRAINER_AUTH`、`TRAINER_COOKIE_SECURE`；一代专用变量归入"遗留"段。
- `oss-release-gate`：回填作者五项决策（0.1–0.5），新增一项待拍板（0.6），
  按实测新增任务 2.8 / 2.9 / 2.10 / 4.7，已完成项打勾。37 个任务，11 个完成。
- 本记录只覆盖文档与配置；业务代码零改动。

## 全新环境实测（按 README 逐条执行）

在 `/tmp/trainer-smoke` 用 `git archive` 导出后从零执行，独立数据库与端口，不触碰线上服务。

| 步骤 | 结果 |
| --- | --- |
| `uv venv` + `uv pip install -r requirements.txt`（清华镜像） | 22 秒，exit 0 |
| `python -m src.cli init-db` | 成功 |
| `npm install`（npmmirror）+ `npm run build` | 12 秒，exit 0 |
| 启动后 `/`、`/login`、`/assets/*` | 均 200 |
| 注册 → `/api/me` → `/api/dashboard` | 均 200 |
| `pytest --collect-only` | 429 条；抽跑两个文件 23 passed |
| **从零到启动合计** | **35 秒**（依赖缓存已热；镜像对速度影响见下） |

## 实测抓到的问题（均已写入 `oss-release-gate`）

1. **`python -m venv` 在本机直接失败**：系统缺 `python3.12-venv`。README 改为以 uv 为主路径、
   venv 作折叠备选并附 `apt install python3-venv` 提示。
2. **直连 PyPI 超时**：本机 6 秒超时，清华镜像 0.29 秒；npm 官方源 4.3 秒，国内镜像 0.75 秒。
   首轮实测卡在安装步骤 13 分钟后被手动终止。README 增加"中国大陆网络"镜像提示。
3. **个人免登录模式不可用**：`TRAINER_AUTH=0` 时 `POST /api/wizard/start` 返回 401。
   根因：`wizard_start` 调 `_require_user(viewer)`，全仓共 18 处端点硬性要求登录用户，
   而免登录模式下没有用户。线上一直是登录模式所以没暴露。README 不再推荐该模式；
   去留列为待拍板 0.6，推荐废弃。
4. **`.env.example` 的占位 key 非空**：`sk-your-api-key-here` 会让程序以为已配置，
   每次模型调用真的去请求三次并吃三次 401，而不是立即降级。改为留空后，
   日志变为即时的 `DEEPSEEK_API_KEY is not set`。
5. **`TRAINER_TZ` 写进 `.env` 不生效**：`src/core/plan.py` 在模块导入时读该变量，
   早于 `load_dotenv()`。`.env` 里写 `UTC` 得到的仍是 `Asia/Shanghai`；直接设在环境变量里才生效。
   `.env.example` 暂不收录该变量，README 表格里如实标注。
6. **接口文档未受保护**：`/docs`、`/openapi.json` 不在 `/api` 前缀下，未登录返回 200。

## 空 key 时向导能走多远

`.env` 留空 key、登录模式下：注册 → `wizard/start`（降级为表单）→ `pace` → `draft`（手填等级与
验收）→ `confirm` → `sources`（粘贴文本）→ `probe`（`fallback_used=true`）→ `activate`
（`status=active`）→ `GET /api/today`，全部 200。README"没有 API key"一条据此写成
"向导也能走完，但依赖模型的功能全部走降级路径"。

## 纪律备注

- README 源码里留了一条位置注释（建议放界面截图或 GIF），由作者录制时补。
- 没有链接 `docs/architecture.md`：它的"项目定位"段仍是早期口径，列为任务 2.10。
- 本机网络下只能验证带镜像的安装路径；venv 路径因系统缺包未能实测，README 里如实给出提示。
- 业务代码零改动；未触碰并行会话的文件；提交使用 pathspec，避开其暂存区。
