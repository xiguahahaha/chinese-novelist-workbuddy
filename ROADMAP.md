# Roadmap

> 这里是 **WorkBuddy 适配分支**的路线图：只跟踪与上游不同的部分。
>
> 写作规则本身的功能规划请看[上游 Roadmap](https://github.com/PenglongHuang/chinese-novelist-skill/blob/master/ROADMAP.md)。
>
> 变更记录见 [Releases](../../releases)。

## 已完成

- **2.3.2-wb.1** — 基于上游 `v2.3.2` 重建，只保留 4 处适配改动
  - Agent Teams 落到 WorkBuddy Agent 工具（`team_name` / `SendMessage` / `bypassPermissions` / `max_turns`）
  - 写作模式选择、字数脚本路径改为 `~/.workbuddy/skills/chinese-novelist-workbuddy/`
  - 补充 `.gitignore`
  - 重写 README：fork 声明、兼容性矩阵、FAQ、Mermaid 流程图

## 进行中

- **上游跟随** — 上游每发一版，重新套用 4 处适配并重跑一次一致性校验

## 计划中

- **无 Node 环境的字数校验** — 目前只有 `.mjs` 脚本；若确实有用户环境不带 Node，再评估是否补一个等价实现
- **实操 FAQ 扩充** — 收集 WorkBuddy 下的真实踩坑：超长上下文、并行写入冲突、偏好重置
- **多开与并发** — 同一项目同时开两个会话写作的互斥策略
