---
title: "k8s-expert-agent：一个 Kubernetes 代码审查 Agent"
date: 2026-07-25
draft: false
tags: ["k8s", "ai-agent"]
collections: ["Kubernetes Operators"]
weight: 1
summary: "把 Kubernetes API Conventions 做成可执行的代码审查 Agent。"
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## 这是什么

[k8s-expert-agent](https://github.com/xuezhaojun/k8s-expert-agent) 是一个做 Kubernetes 代码审查的 Agent，本身就是一个 Git 仓库（[repo-as-agent 模式](/posts/repo-as-agent/)）。你把它 clone 下来，从仓库根目录启动自己的 coding agent（Claude Code、Codex、OpenCode 等），它就读 `AGENTS.md` 拿到 K8s 审查专家的身份和能力。

审查规则 follow [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md)，覆盖 CRD / API 设计和 controller 行为。规则会持续补充，逐步把更多最佳实践沉淀进来。

## 怎么用

三步：

```bash
git clone https://github.com/xuezhaojun/k8s-expert-agent.git
cd k8s-expert-agent
git pull origin main   # 保持同步
```

然后直接跟你的 coding agent 对话，给它审查目标，三种都行：

- 一个 **repo 链接** —— `review this operator: https://github.com/me/my-operator`
- 一个 **PR 链接** —— `review this PR: https://github.com/me/my-operator/pull/42`
- 一个 **本地目录** —— `review the controller under ./my-operator/`

再给它具体的 Code Review 指令（比如"重点看 CRD 设计和 status 处理"），它就会去跑审查。意图模糊时它会先问你清楚。

> Claude Code 是个例外：它从 `.claude/skills/` 读技能，需要 `ln -s ../.agent/skills .claude/skills` 软链一次。其他 agent 直接可用。前置依赖只有 Go 工具链和 `curl`。

## 审查流程

拿到目标后，Agent 按几步走：

1. **取代码** —— PR 用 `fetch_pr.sh` 取到临时目录、按 `git diff` 确定审查范围；repo 或本地目录用 `clone_repo.sh`、按完整文件清单确定范围。
2. **跑自动 linter** —— 对 Go API 类型调用 [kube-api-linter](https://github.com/kubernetes-sigs/kube-api-linter)（KAL），机械地抓命名、注释、字段类型这类确定性问题。KAL 的规则直接来自 API Conventions。
3. **校验 CRD 新鲜度** —— 当 Go 类型和 CRD YAML 都在范围内时，用 `controller-gen` 重新渲染 CRD 并和已提交的 YAML diff，避免基于过期 YAML 得出错误结论。
4. **应用 checklist** —— 把 KAL 抓不到的设计判断和 controller 行为问题按 checklist 分章节并行审查，最后整合去重。

最终输出一份**统一的报告**：按 severity（blocking / warning / suggestion）分组，每条 finding 带 `file:line` 和具体修法。

## 依赖的工具

- **[kube-api-linter](https://github.com/kubernetes-sigs/kube-api-linter) (KAL)** —— 自动 lint Kubernetes Go API 类型，是审查的机械层。
- **[controller-tools](https://github.com/kubernetes-sigs/controller-tools) (`controller-gen`)** —— 渲染 OpenAPIV3Schema，用于 CRD 新鲜度校验和跨版本兼容性 diff。
- **golangci-lint v2** —— 用来构建自定义的 KAL 二进制。

所有版本 pinned 在 `tools/versions.env`，审查结果可复现。

## 欢迎试用

如果你在写 controller 或 operator，可以用它来跑一遍 Code Review，看看自己的 CRD 设计和 controller 实现是否满足 Kubernetes 的最佳实践。规则和脚本都接受 PR，欢迎一起补充。
