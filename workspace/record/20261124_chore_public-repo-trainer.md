# 20261124 · chore · 公开仓落地——新建全新历史的 trainer 并推送，从 GitHub 克隆复测通过

## 变更摘要

- 新建公开仓 **`q1ngn1ng-web/trainer`**（<https://github.com/q1ngn1ng-web/trainer>）：
  全新历史、默认分支 `main`、单个初始提交、488 个文件；MIT 许可证被 GitHub 正确识别；
  Issues 已开启；话题标签 9 个。
- 旧的 `personal_trainer_agent` 按作者决定保持公开、不做任何改动。
- 来源：本地仓 `006acdf` 之后的文档提交，经白名单导出后推送。
- README 补一句中国大陆网络下的克隆提示（HTTPS 克隆在本机实测卡住，SSH 正常）。
- `oss-release-gate` 新增待拍板 0.7（开发主仓与公开仓的关系）与任务 2.11（评测脚本的绝对路径）。

## 导出白名单与排除项

带进公开仓的：源码、前端、测试、脚本、文档、规格与归档（openspec）、决策记录（ADR）、
**个人开发记录**（`workspace/record/` 93 篇，作者 2026-11-23 拍板）、`BACKLOG.md`、`HANDOFF.md`、
评测报告（`workspace/evidence/` 除下列四个文件）。

**没有带进去的**：

| 路径 | 理由 |
| --- | --- |
| `workspace/面试/`（7 个文件） | 求职与面试材料 |
| `workspace/evidence/判分一致性标注-{工作表,工作表-真实数据}.csv`、`-人工填写.md`、`-练习集.md` | 含真实作答与个人盲标 |
| `data/Redis 持久化八股文.pdf` | 个人学习资料 |
| `workspace/prompts/`、`AUDIT-BRIEF.md`、`SUMMARY-*.md` | 个人工作提示词与汇总 |
| `.codegraph/` | 工具本地状态 |

这些文件在规格与记录里被引用，公开仓中会是悬空引用——属于已知且可接受的代价。

## 推送前的隐私扫描（导出目录 488 个文件）

| 类别 | 结果 |
| --- | --- |
| API key 形态 / 测试口令 / 手机号 / Cookie 值 | 全部 0 处 |
| 邮箱 74 处 | 域名仅 `example.com`、`personal.test`、`x.com`，均为测试夹具 |
| IPv4 12 处 | 仅 `0.0.0.0`、`127.0.0.1`、`169.254.169.254` 等，无服务器地址 |
| 账号名 33 处 | GitHub ID `q1ngn1ng-web` 与记录里的测试账号名，本身公开 |
| `/home/ubuntu` 绝对路径 12 处 | 4 个评测脚本写死路径，不敏感，列入任务 2.11 |

## 从 GitHub 克隆复测

| 步骤 | 结果 |
| --- | --- |
| `git clone`（SSH） | 46 秒；1 个提交、488 个文件 |
| 不该出现的路径（面试、PDF、真实数据、`.env`、数据库、node_modules） | 全部 0 |
| 依赖（清华镜像）→ `init-db` → 前端构建（npmmirror） | 均 exit 0 |
| `/`、`/login`、`/api/health` | 200 / 200 / `{"status":"ok"}` |
| 注册 → `POST /api/wizard/start`（无 key） | 200 / 200（降级） |
| **克隆到页面可访问合计** | **55 秒** |

## 遗留问题

- **两个仓库从现在起分叉。** 本地开发仓的 `origin` 仍指向旧仓，并行会话照常往旧仓推送；
  `trainer` 只是快照。后续怎么同步是待拍板 0.7。在那之前，不要把本地完整历史 push 到 `trainer`。
- 初始提交的作者身份用的是 GitHub 的 noreply 邮箱（提交会关联到账号头像），没有沿用本地仓里
  邮箱域为 `local` 的占位身份。

## 踩坑

- `git ls-remote https://github.com/...` 在本机卡住超过 100 秒，SSH 秒回——涉及 GitHub 的
  git 操作一律走 SSH。
- `gh repo view --json` 没有 `hasIssuesOptEnabled` 这个字段，正确的是 `hasIssuesEnabled`。
- 用 `pkill -f "uv pip install ..."` 清理进程时，模式匹配到了执行它的那个 shell 自身，后面的命令全没跑。
  清理进程用 `pgrep -af "[u]v pip"` 这类首字符加方括号的写法，并且别和后续命令串在一起。
