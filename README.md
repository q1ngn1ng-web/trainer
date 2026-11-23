<div align="center">

# Trainer

**导入你自己的资料，让 AI 按资料出题、按要点判分，并记住你错在哪。**

个人训练教练 · Python（FastAPI）+ Vue 3 + SQLite

[快速开始](#快速开始) · [它是什么](#它是什么) · [架构](#架构) · [参与贡献](#参与贡献)

</div>

> 仓库名里带着 “agent”，但它**现在不是 agent**：没有自主规划，也没有工具调用，
> 而是一条由规则编排、中间嵌着若干次模型调用的固定流程。为什么这样设计、接下来怎么演进，
> 见[它是什么](#它是什么)与[路线图](#路线图)。

<!-- 建议在这里放一张「今日任务」页的截图或一段 20 秒 GIF：比任何文字都更能让人留下。 -->

## 它是什么

学一样东西时常见三个断点：资料收藏了不读；读了不知道自己会不会；复习全凭感觉。
Trainer 把它们接成一条闭环：

1. **说清目标**：一句话描述想学什么，AI 起草目标等级和验收标准，你改到满意为止。你明确填过的值，AI 不会覆盖。
2. **导入资料**：上传 PDF / Word / Excel / Markdown / TXT，直接粘贴文本，或给一个网址。
3. **摸底**：每个知识点出一道题，定位你处在“已掌握 / 边缘 / 未达”哪一档。
4. **生成训练路径**：已掌握的跳过，处在边缘的作为重点，未达的先补前置。
5. **日常训练**：每天系统按你的资料出题，你作答，系统判分；答错了可以看引用资料和历次作答的解析。
6. **定期测验与复盘**：冷却期后用变式题检验，周复盘给出校准建议。

### 三个设计取舍

| 取舍 | 做法 | 为什么 |
|---|---|---|
| 模型举证，规则判决 | 模型只对每条要点给出“命中 / 部分 / 说反 / 遗漏”，并且必须摘录你的原话作证据；分数与是否通过由规则层计算 | 判定可审计、可复现，换模型也不会悄悄改变标准 |
| 记忆是结构化状态 | 不保存聊天记录；每次作答落库为事实，掌握度与误解清单由作答重放得出 | 记忆可查询、可重建，不依赖上下文窗口 |
| 调用全量可审计 | 所有模型调用走同一个出口，成功、降级、失败都记录提示词版本、耗时与 token | 每个结果都能追到“哪一版提示词、哪个模型、花了多少” |

### 还有一些

- 选择题、填空题走确定性判分，不调用模型
- 学习者模型：每个知识点的掌握度、随时间衰减、活跃误解清单
- 仪表盘、训练热力图、知识点覆盖、会话小结、连续答错提醒
- 训练与资料三档可见性、分享链接导入副本、组织与成员角色
- 深色 / 浅色 / 跟随系统三种主题

### 它不是什么

- **不是 agent。** 见文首说明。
- **不是 RAG 问答。** 没有向量检索；资料用来让出题和解析“有据可依”，不用于回答自由提问。
- **不是背单词 App 的替代品。** 复习间隔目前是固定的经验常量，见[证据边界](#证据边界)。

## 快速开始

需要：Python 3.12、Node.js 20 或更高，以及一个兼容 OpenAI `/chat/completions` 的模型服务（默认 DeepSeek；
没有 key 也能启动，见[常见问题](#常见问题)）。

```bash
git clone https://github.com/q1ngn1ng-web/trainer.git
cd trainer

# 1) 后端依赖（需要 uv：https://docs.astral.sh/uv/ ，已有的话 `pip install uv` 也行）
uv venv && source .venv/bin/activate
uv pip install -r requirements.txt

# 2) 配置模型
cp .env.example .env      # 编辑 .env，填入 DEEPSEEK_API_KEY

# 3) 初始化数据库
python -m src.cli init-db

# 4) 前端
cd web && npm install && npm run build && cd ..

# 5) 启动
uvicorn src.api.app:app --port 8501
```

浏览器打开 <http://127.0.0.1:8501>，注册一个账号即可使用——账号和数据只存在你本地的 SQLite 里。
请在项目根目录执行启动命令，`.env` 才会被读到。

**在中国大陆网络下**，直连 PyPI 和 npm 官方源可能极慢甚至超时，克隆也可能很慢。克隆慢时改用 SSH 地址
`git@github.com:q1ngn1ng-web/trainer.git`；两条安装命令加上镜像：

```bash
uv pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
cd web && npm install --registry=https://registry.npmmirror.com && npm run build && cd ..
```

<details>
<summary>不想装 uv：用 Python 自带的 venv</summary>

把第 1 步换成：

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Ubuntu 上如果报 `ensurepip is not available`，先执行 `sudo apt install python3-venv`。

</details>

### 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `DEEPSEEK_API_KEY` | 无 | 模型服务的 API key，**必填** |
| `DEEPSEEK_BASE_URL` | `https://api.deepseek.com` | 模型服务地址。按接口约定可换成任何兼容 `/chat/completions` 的服务，未逐一验证 |
| `DEEPSEEK_MODEL` | `deepseek-v4-flash` | 模型名 |
| `TRAINER_AUTH` | `1` | `1` 必须登录。`0` 是早期遗留的免登录模式，目前新建训练会失败，不建议使用 |
| `TRAINER_COOKIE_SECURE` | `0` | 用 HTTPS 部署时设为 `1` |
| `TRAINER_TZ` | `Asia/Shanghai` | “今天”按哪个时区计算。要直接设在环境变量里，写进 `.env` 目前不生效 |
| `DB_PATH` | `./data/trainer.db` | SQLite 文件位置 |

### 常见问题

- **打开首页是 404**：还没有构建前端。后端只在 `web/dist` 存在时才提供页面，先完成第 4 步。
- **接口报 `no such table`**：没有执行 `python -m src.cli init-db`。
- **没有 API key**：服务能启动，向导也能走完，但依赖模型的功能全部走降级路径——目标澄清变成需要你自己填的
  表单，摸底和出题退回通用问法，主观题不作判定。
- **换端口**：改 `--port` 即可。页面与接口由同一个进程提供，不需要额外配置。

## 架构

一次日常作答经过的环节：

```
今日任务 ──► 系统出题（依据你的资料切片）──► 你作答
                                                │
          规则层算分 ──► 作答记录（唯一事实源）◄─┘ 模型逐条判定要点
             │                  │
       是否通过、掌握度     学习者模型（可由作答重放重建）
```

```
src/
├── api/app.py      全部 HTTP 接口与登录鉴权中间件
├── core/           纯函数领域逻辑：排期、掌握度、会话配比、权限判定（不碰数据库与模型）
├── services/       业务编排：出题、判分、解析、计划、记忆、分享……
├── llm/            统一模型客户端、提示词与 JSON Schema 注册表、重试与降级
├── db/             SQLite：建表语句、幂等迁移、查询
└── main.py、ui/    已下线的一代界面（Streamlit），正在移除，请勿在此基础上开发
web/                Vue 3 + TypeScript + Tailwind CSS 前端
tests/              后端测试
scripts/            评测与冒烟脚本
workspace/
├── store/openspec/ 规格与变更提案
└── decisions/      架构决策记录
docs/               架构说明与训练理论资料
```

## 测试

```bash
pytest tests/ -q          # 全量约 6 分钟
cd web && npm run build   # 前端目前没有自动化测试，以构建通过为准
```

## 证据边界

在用它说服自己（或别人）之前，先知道它的边界：

- **判分一致性只在很小的样本上测过。** 人工盲标 6 条要点判定：按严格的四态口径，与模型完全一致的是
  0 条；放宽到“答到 / 没答到”二值是 4 条。分歧集中在“答到什么程度算数”这条边界上。判分口径仍在校准，
  欢迎参与标注。
- **复习间隔是经验常量。** 固定五轮（第 1 / 3 / 7 / 15 / 30 天）、冷却 14 天、掌握度半衰期 30 天，
  对所有人一样。用遗忘曲线来调度在计划中。
- **没有学习效果的对照数据。** 目前没有任何“用了之后学得更好”的证据。

## 路线图

以下是计划，不是承诺；每一项在 `workspace/store/openspec/changes/` 里都有对应的提案。

- **近期**：开源发布门禁（依赖、文档、公网部署护栏）；移除一代遗留代码
- **中期**：教练对话（把现有服务封装成模型可调用的工具）；用遗忘曲线调度替代固定间隔；出题前的资料检索

## 参与贡献

这是一个边用边做的个人项目，欢迎从任何一处入手。

- **入门任务**：Issues 里会陆续标出 `good first issue`，每个都带验收标准。
- **先看规格再动手**：新功能先在 `workspace/store/openspec/changes/` 写提案，再实现、再归档。
  面向 AI 协作者的流程约定见仓库根目录的 `AGENTS.md`。
- **测试要求**：涉及可见性、权限、过滤的改动，请附上走 HTTP 接口的用例和跨用户隔离用例。
  这个项目多次踩过“函数对了、接线断了、测试全绿”的坑。
- **提交信息**：`类型(范围): 中文摘要`，类型取 feat / fix / docs / refactor / chore。

## License

[MIT](LICENSE)
