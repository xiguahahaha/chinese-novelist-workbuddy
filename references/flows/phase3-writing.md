# 第三阶段：疯狂创作

**重要：全程无需再次向用户确认，必须逐章创作直到完成**

---

## 0. 启动检测与模式读取

开始创作前：
1. 读取 `02-写作计划.json`
2. 读取 `writingMode` 字段，进入对应模式流程
3. 如果存在 `status: "in_progress"` 的章节 → 从该章节继续（中断续写）
4. 如果所有章节 `status: "pending"` → 从第 1 章开始
5. 如果存在 `status: "failed"` 的章节（Phase 4 回退）→ 从第一个 failed 章节开始
6. 如果项目缺少 `03-文风基准.md`（旧版本项目）→ 已有完成章节的，从第 1 章提取最佳段落补生成；一个章节都没有的，按题材生成默认基准
7. **旧项目迁移**：如果 `01-大纲.md` 存在非空「章节摘要」区且 `99-章节摘要.md` 不存在 → 将摘要区内容迁移到 `99-章节摘要.md`，并删除大纲中的「章节摘要」区

---

## 1. 逐章创作流程（所有模式共用）

每一章的完整创作流程（写前分析 → 撰写 → 撰写后优化 → 收尾）统一执行 **[chapter-checklist.md](chapter-checklist.md)**，所有写作模式共用同一份清单，不得自行增删步骤。

---

## 2. 串行模式（writingMode: "serial"）

**主 Agent 自己逐章创作，全程不中断。**

### 自驱循环

```
WHILE 02-写作计划.json 中存在 status != "completed" 的章节:
    执行 [chapter-checklist.md](chapter-checklist.md) 的完整流程（阶段 A-D）
    ⚠️ 完成一章后，立即读取 JSON 认领下一章，不要向用户确认，不要停下来
所有章节完成 → 进入第四阶段：自动校验
```

**关键提醒（每章完成后必须遵守）**：
> 本章已完成。立即读取 `02-写作计划.json`，认领下一个 pending 章节，开始下一章创作。不要使用 AskUserQuestion，不要向用户确认，不要停下来。你必须把所有章节创作完成才能与用户报告。

---

## 3. 子Agent并行模式（writingMode: "subagent-parallel"）

**核心机制**：主 Agent 将章节分成不重叠的批次，每个批次派生一个子 Agent。批次内串行写作，批次间并行执行。

### 主 Agent 流程

```
1. 计算批次分配:
   - 每批 5-8 章（根据总章数动态调整）
   - 批次间不重叠
   - 示例: 30 章 → 5 批 × 6 章
2. 为每个批次派生子 Agent（使用 Agent 工具，多个批次可并行派生）:
   - 每个 Agent 内部串行执行 [chapter-checklist.md](chapter-checklist.md) 的完整流程
   - 主 Agent 派生时必须将 prompt 中的 {skillBaseDir} 替换为 skill 安装根目录的绝对路径
3. 所有子 Agent 完成后 → 进入第四阶段：自动校验
```

### 子 Agent prompt 模板（并行模式）

> 派生前将 `{skillBaseDir}` 替换为 skill 安装根目录的绝对路径，`{start}`/`{end}` 替换为批次章节范围。

```
你是一个小说批量创作 Agent。你需要创作第 {start} 章到第 {end} 章。

## 项目信息
- 项目路径: {projectPath}
- 你负责的章节: 第 {start} 章 到 第 {end} 章

## 创作步骤
对每一章依次执行 {skillBaseDir}/references/flows/chapter-checklist.md 的完整流程（阶段 A-D），不得跳过任何步骤。

## 重要约束
- 不要使用 AskUserQuestion，不要向用户确认任何事
- **每章开始前必须读取大纲**，严格按大纲的核心事件和悬念钩子创作
- 字数必须达到 3000 字以上
- 共享文件（02-写作计划.json、99-章节摘要.md、设定词典）只用 Edit 工具更新且改前重读，禁止 Bash 整体重写
- 你负责的所有章节必须全部完成

完成后报告: 各章编号、字数、是否通过字数检查
```

### 并行写入协议

写作工具对并发修改有检测（文件在读取后被他人改动，Edit/Write 会报错强制重读），但 Bash 写入没有这层保护，报错重试也会浪费轮次。因此并行期间各批次必须遵守：

1. **写作计划**：每批只允许用 Edit 工具修改 `02-写作计划.json` 中**自己章节**的 `status`/`wordCount`/`wordCountPass` 字段；修改前必须重新读取 JSON（不要基于派生时缓存的旧内容 Edit）；**禁止用 Bash/脚本整体重写 JSON**
2. **章节摘要**：按各自章号小节追加到 `99-章节摘要.md`，同样只用 Edit 且改前重读
3. **设定词典**：更新 `01-大纲.md` 设定词典只用 Edit 且改前重读；遇到持续报错，稍候重试，不要改用 Bash 追加
4. **跨批次衔接**：每批第一章动笔时，上一批末章可能尚未落盘——此时以大纲该章「承接上章」「悬念钩子」栏为衔接依据，不阻塞等待。全部批次完成后，主 Agent 对每个批次接缝做衔接检查（读前批末章结尾 + 后批首章开头），断裂处轻量修顺

---

## 4. 多 Agent 协作模式（writingMode: "agent-teams"）

**通用描述**：适用于宿主环境提供多 Agent 协作机制的场景（共享任务队列、Agent 间通讯、团队管理）。**以能力描述为准，具体工具名称随宿主环境而定**——环境中有什么就用什么；不提供这些能力时，回退到「子Agent并行模式」。（WorkBuddy 下直接用 Agent 工具的团队模式：`team_name` 组队、`SendMessage` 实现成员间通讯与任务认领。）

### 主 Agent 流程（WorkBuddy 实现）

```
1. 选定团队名 teamName（同一会话内唯一，例如 "novel-team-{timestamp}"）
2. 并行派生 3-5 个团队成员（每个成员调用一次 Agent 工具）：
   - subagent_type: "general-purpose"
   - name:        唯一，例如 writer-1 / writer-2 / writer-3 …（同一团队内不可重名）
   - team_name:   上面的 teamName（同队成员可用 SendMessage 互相协调）
   - mode:        "bypassPermissions"（写作全程无需逐次确认）
   - max_turns:   适当上限（如 40），防止单章失控
   - prompt:      下方「团队成员 prompt 模板」（把 {skillBaseDir}/{projectPath}/{teamName} 替换为实际值）
3. 成员按模板循环认领 02-写作计划.json 中的 pending 章节，各自执行 chapter-checklist.md
   - 认领 = 用 Edit 把该章 status 改为 "in_progress"（改前先重读 JSON，避免覆盖他人改动）
   - 完成 = status 改 "completed"，写入 wordCount，并按 chapter-checklist.md 阶段 D 更新 99-章节摘要.md
4. 所有章节均为 completed → 主 Agent 调用「第四阶段：自动校验」
```

> 若当前环境不提供多 Agent 团队能力，回退到「子Agent并行模式」即可，创作质量不受影响。

### 团队成员 prompt 模板（作为 Agent 工具的 prompt 参数直接下发）

```
你是一个小说创作团队成员，属于团队「{teamName}」。按团队任务机制认领章节并创作，全程不要向用户确认。

## 项目信息
- 项目路径: {projectPath}
- 逐章清单: {skillBaseDir}/references/flows/chapter-checklist.md（阶段 A-D 不得跳步）
- 进度与任务队列: {projectPath}/02-写作计划.json（认领 / 完成都改这里）

## 工作循环
1. 读取 02-写作计划.json，找出一个 status="pending" 且未被他人认领的章节
2. 用 Edit 工具把该章 status 改为 "in_progress"（改前先重读 JSON，避免覆盖他人改动）
3. 对该章执行逐章清单完整流程（阶段 A-D），每章开始前必须读取 01-大纲.md 对应章节
4. 完成后把 status 改为 "completed"，写入 wordCount；并按 chapter-checklist.md 阶段 D 的写法更新 99-章节摘要.md 中本章记录
5. 认领下一章，直到没有 pending 章节；全部完成后向主 Agent 汇报各章编号、字数、是否通过字数检查

## 重要约束
- 不要向用户确认任何事
- **每章开始前必须读取大纲**，严格按大纲的核心事件和悬念钩子创作
- 字数必须达到 3000 字以上
- 只写自己认领（in_progress）的章节，禁止改他人 in_progress 的章节
- 共享文件（写作计划 JSON、章节摘要、设定词典）写入遵守「并行写入协议」：只用 Edit 且改前重读，禁止 Bash 整体重写
- 如需与其他成员协调，用团队通讯（SendMessage 给 team_name 中的成员），不要写他人已认领的章节
```

**并发安全**：任务认领的原子独占语义保证一章只被一个成员写；共享文件（写作计划 JSON、章节摘要、设定词典）的写入遵守「并行写入协议」。
