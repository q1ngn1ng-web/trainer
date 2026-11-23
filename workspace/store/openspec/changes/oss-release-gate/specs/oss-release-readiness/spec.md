# oss-release-readiness Specification

## ADDED Requirements

### Requirement: 仓库入口文档必须与现实一致

README 的 Quick Start SHALL 能让陌生人在全新环境里启动当前产品（FastAPI + Vue），
MUST NOT 指向已被取代的 Streamlit 一代界面。README MUST 列出启动所需的全部环境变量及
其默认值，并如实说明项目当前的证据边界（例如判分一致性人工标注的样本量）。
随仓库提供的配置模板 MUST 与代码默认值一致，且 MUST NOT 以非空占位值充当密钥。
项目根的 AGENTS.md 对入口、技术栈、目录职责的描述 MUST 与代码一致。

#### Scenario: 陌生人按 README 启动

- **WHEN** 在一台只装了 Python 3.12 与 Node 22 的新机器上，从 `git clone` 起逐条执行 README 的 Quick Start
- **THEN** 10 分钟内能在浏览器看到 Vue 登录页并完成注册
- **THEN** 未配置 `DEEPSEEK_API_KEY` 时，页面给出明确的缺 key 提示，而不是白屏或 500

#### Scenario: 文档不再指向一代界面

- **WHEN** 在 README 与 AGENTS.md 中搜索 `streamlit run`
- **THEN** 零命中，或仅出现在明确标注"已下线"的说明里

#### Scenario: 配置模板不留占位密钥

- **WHEN** 用户把 `.env.example` 原样复制为 `.env`（不填密钥）后启动
- **THEN** 服务正常启动，且不向模型服务发起任何请求；依赖模型的功能立即走降级路径

### Requirement: 环境变量按文档所述方式生效

README 与 `.env.example` MUST 说明每个环境变量的默认值；写进 `.env` 即可生效的变量与必须直接设在
环境里的变量 MUST 区分标明。缺陷修复之前，MUST 如实标注"写进 `.env` 不生效"。

#### Scenario: 运行模式写进 .env

- **WHEN** `.env` 里写 `TRAINER_AUTH=0` 后启动
- **THEN** `GET /api/auth/mode` 返回 `auth_enabled=false`

#### Scenario: 时区变量的标注与现状一致

- **WHEN** `.env` 里写 `TRAINER_TZ=UTC` 后启动
- **THEN** 若 README 标注该变量写进 `.env` 不生效，实际时区保持默认值；修复后实际时区为 UTC，且该标注随之移除

### Requirement: 依赖清单可安装

`requirements.txt` SHALL 覆盖启动与测试所需的全部直接依赖（含 fastapi、uvicorn、
python-multipart、pytest），且不含重复项。

#### Scenario: 全新虚拟环境安装

- **WHEN** 在空的虚拟环境里执行 `pip install -r requirements.txt`
- **THEN** `python -c "import src.api.app"` 与 `python -m pytest --collect-only -q` 均成功

### Requirement: 仓库携带开源许可证

仓库根目录 SHALL 有 LICENSE 文件，README 声明同一许可证。许可证类型由作者拍板。

#### Scenario: 访问者可以确认授权条款

- **WHEN** 访问者打开仓库根目录
- **THEN** 能看到 LICENSE 文件，且 README 的 License 一节与之相符

### Requirement: 私人材料不进入公开仓库

公开仓库 MUST NOT 跟踪：作者的求职与面试材料、含真实个人作答数据的数据集、
`data/` 下的个人学习资料文件、个人工作提示词与个人工作汇总文档。
公开仓库的 git 历史 MUST NOT 含有 API 密钥形态的字符串、账号口令、`.env` 文件或数据库文件。

#### Scenario: 公开仓库索引检查

- **WHEN** 对公开仓库执行 `git -c core.quotepath=false ls-files`
- **THEN** 结果中不含 `workspace/面试/`、文件名含「真实数据」的工作表、`data/*.pdf`、`workspace/prompts/`

#### Scenario: 历史密钥扫描

- **WHEN** 对公开仓库的全部提交执行密钥形态扫描（如 `sk-[A-Za-z0-9]{32,}`）
- **THEN** 零命中

### Requirement: 公网部署有基本护栏

当实例可被公网访问时：注册 MUST 可通过配置关闭；登录、注册与触发 LLM 调用的端点
MUST 按来源限流，超限返回 429；会话 Cookie 在 HTTPS 下 MUST 标记 Secure；
每个用户的每日 LLM 调用量 MUST 有上限。本地 HTTP 调试的默认配置下，行为 MUST 与
引入护栏之前一致。未捕获的服务端异常 SHALL 以 JSON 返回并带请求追踪 id。
开启登录（`TRAINER_AUTH=1`）时，接口文档端点 MUST NOT 对外开放。

#### Scenario: 关闭注册

- **WHEN** `TRAINER_REGISTER_OPEN=0`，未登录用户 `POST /api/auth/register`
- **THEN** 返回 403 与可读提示；既有账号登录不受影响

#### Scenario: 限流

- **WHEN** 同一来源 IP 在窗口期内对登录端点发起的请求数超过阈值
- **THEN** 超出部分返回 429 并带 `Retry-After`

#### Scenario: 成本熔断

- **WHEN** 某用户当日 LLM 调用数达到上限后再次触发日常出题
- **THEN** 返回可读的错误说明（今日额度已用完），不调用 LLM，`plan_items.question` 保持为空

#### Scenario: 开启登录时不暴露接口文档

- **WHEN** `TRAINER_AUTH=1`，未登录访问 `/docs`、`/redoc` 或 `/openapi.json`
- **THEN** 返回 404；`TRAINER_AUTH=0` 的本地模式下这三个地址照常可用

#### Scenario: 默认配置行为不变

- **WHEN** 按默认配置在本地 HTTP 启动且未设置任何护栏相关环境变量
- **THEN** 注册、登录、出题、判分的行为与引入护栏之前一致
