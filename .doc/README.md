# sibot 文档

> 项目定位待定（TODO）。这里先作为所有设计与决策文档的落点。

## 仓库规则（先读这个）

| 文件 | 作用 |
|------|------|
| [`CONTRIBUTING.md`](../CONTRIBUTING.md) | 提交内容禁区、分支、commit 规范、PR 规范、AI 辅助声明 |
| [`SECURITY.md`](../SECURITY.md) | 漏洞私密上报流程 |
| [`.github/pull_request_template.md`](../.github/pull_request_template.md) | PR 模板（自动带出） |
| [`.gitignore`](../.gitignore) | 密钥/产物防误提交第一道防线 |

一句话摘要：**公开仓库、历史不可撤回** → 密钥/个人信息/雇主内部信息零容忍；commit 用
`type(scope): subject`（Conventional Commits，英文，≤72 字符）；一个 commit 一个逻辑变更；
用了 AI 就在 trailer 里写 `Co-Authored-By` 并给 PR 打 `ai-assisted`。

## 目录约定

| 文件 | 内容 |
|------|------|
| `README.md` | 本索引 |
| `requirements.md` | 需求 / 用户故事 / 范围（待写） |
| `design.md` | 架构与关键设计决策（待写） |
| `decisions/` | ADR，一篇一个决策 `NNNN-<slug>.md`（待建） |

## 待定问题

- [ ] sibot 解决什么问题？给谁用？
- [ ] 输入 / 输出是什么（CLI、常驻服务、Slack/Jira 集成、agent？）
- [ ] 技术栈与部署形态
- [ ] LICENSE 选型（目前 `LICENSE` 文件还不存在）
