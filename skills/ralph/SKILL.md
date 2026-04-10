---
name: ralph
description: "将 PRD 转换为 Ralph 自主代理系统使用的 prd.json 格式。适用于你已经有现成 PRD，需要把它转成 Ralph JSON 格式的场景。触发词包括：convert this prd、turn this into ralph format、create prd.json from this、ralph json。"
user-invocable: true
---

# Ralph PRD 转换器

将现有 PRD 转换为 Ralph 用于自主执行的 `prd.json` 格式。

---

## 任务目标

接收一个 PRD（markdown 文件或纯文本），并将其转换为你 ralph 目录下的 `prd.json`。

---

## 输出格式

```json
{
  "project": "[项目名称]",
  "branchName": "ralph/[feature-name-kebab-case]",
  "description": "[从 PRD 标题或引言提取的功能描述]",
  "userStories": [
    {
      "id": "US-001",
      "title": "[Story 标题]",
      "description": "作为 [user]，我希望 [feature]，以便 [benefit]",
      "acceptanceCriteria": [
        "标准 1",
        "标准 2",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Story 大小：最重要的规则

**每条 story 都必须能在 ONE 次 Ralph 迭代中完成（也就是一个上下文窗口内完成）。**

Ralph 每轮都会启动一个全新的 Amp 实例，不会保留前一轮的工作记忆。如果某条 story 太大，LLM 会在完成前耗尽上下文，并产出有问题的代码。

### 合适大小的 story：
- 添加一个数据库字段和 migration
- 在现有页面中增加一个 UI 组件
- 修改一个 server action 的逻辑
- 给列表增加一个筛选下拉框

### 太大的任务（应继续拆分）：
- “Build the entire dashboard” 应拆成：schema、queries、UI components、filters
- “Add authentication” 应拆成：schema、middleware、login UI、session handling
- “Refactor the API” 应按 endpoint 或模式拆成多条 story

**经验法则：** 如果你无法用 2 到 3 句话描述清楚改动，那它通常就太大了。

---

## Story 排序：依赖优先

Stories 会按优先级顺序执行。靠前的 story 不能依赖靠后的 story。

**正确顺序：**
1. Schema / 数据库变更（migrations）
2. Server actions / 后端逻辑
3. 使用后端能力的 UI 组件
4. 汇总展示数据的 dashboard / summary 页面

**错误顺序：**
1. UI 组件（依赖尚不存在的 schema）
2. Schema 变更

---

## 验收标准：必须可验证

每条标准都必须是 Ralph 可以 CHECK 的内容，不能是模糊表述。

### 好的标准（可验证）：
- "Add `status` column to tasks table with default 'pending'"
- "Filter dropdown has options: All, Active, Completed"
- "Clicking delete shows confirmation dialog"
- "Typecheck passes"
- "Tests pass"

### 不好的标准（模糊）：
- "Works correctly"
- "User can do X easily"
- "Good UX"
- "Handles edge cases"

### 结尾必须包含：
```
"Typecheck passes"
```

对于可测试逻辑的 story，还应包含：
```
"Tests pass"
```

### 对于涉及 UI 变更的 story，还应包含：
```
"Verify in browser using dev-browser skill"
```

前端 story 在完成视觉验证之前都不算完成。Ralph 会使用 dev-browser skill 打开页面、与 UI 交互，并确认改动确实生效。

---

## 转换规则

1. **每个 user story 对应一个 JSON 条目**
2. **ID**：按顺序递增（US-001、US-002 等）
3. **Priority**：先按依赖顺序，再按文档顺序
4. **所有 story**：初始都设为 `passes: false`，`notes` 为空
5. **branchName**：从功能名推导，使用 kebab-case，并加上 `ralph/` 前缀
6. **始终补充**：每条 story 的验收标准里都要有 `"Typecheck passes"`

---

## 拆分大型 PRD

如果一个 PRD 包含很大的功能块，就要拆分：

**原始需求：**
> "Add user notification system"

**可拆分为：**
1. US-001: 在数据库中添加 notifications 表
2. US-002: 创建发送通知的 notification service
3. US-003: 在 header 中加入通知铃铛图标
4. US-004: 创建通知下拉面板
5. US-005: 添加已读标记功能
6. US-006: 添加通知偏好设置页面

每一条都是一个聚焦的改动，可以独立完成、独立验证。

---

## 示例

**输入 PRD：**
```markdown
# 任务状态功能

为任务增加不同状态标记的能力。

## 需求
- 在任务列表中切换 pending / in-progress / done
- 按状态筛选列表
- 为每个任务显示状态徽章
- 状态持久化到数据库中
```

**输出 prd.json：**
```json
{
  "project": "TaskApp",
  "branchName": "ralph/task-status",
  "description": "任务状态功能 - 通过状态标识跟踪任务进度",
  "userStories": [
    {
      "id": "US-001",
      "title": "为 tasks 表添加状态字段",
      "description": "作为开发者，我需要把任务状态存储到数据库中。",
      "acceptanceCriteria": [
        "添加状态字段：'pending' | 'in_progress' | 'done'（默认值为 'pending'）",
        "成功生成并运行 migration",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-002",
      "title": "在任务卡片上显示状态徽章",
      "description": "作为用户，我希望一眼就能看到任务状态。",
      "acceptanceCriteria": [
        "每张任务卡片都显示彩色状态徽章",
        "徽章颜色：gray=pending，blue=in_progress，green=done",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "priority": 2,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-003",
      "title": "在任务列表行中添加状态切换控件",
      "description": "作为用户，我希望可以直接在列表中修改任务状态。",
      "acceptanceCriteria": [
        "每一行都有状态下拉框或切换控件",
        "修改状态后立即保存",
        "无需刷新页面即可更新 UI",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "priority": 3,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-004",
      "title": "按状态筛选任务",
      "description": "作为用户，我希望筛选列表，只查看特定状态的任务。",
      "acceptanceCriteria": [
        "筛选下拉框：All | Pending | In Progress | Done",
        "筛选条件持久化到 URL 参数中",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "priority": 4,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## 归档之前的运行结果

**在写入新的 prd.json 之前，先检查是否已经存在一个来自其他功能的旧 prd.json：**

1. 如果当前存在 `prd.json`，先读取它
2. 检查其中的 `branchName` 是否与新功能的分支名不同
3. 如果不同，且 `progress.txt` 除表头外已有内容：
   - 创建归档目录：`archive/YYYY-MM-DD-feature-name/`
   - 把当前的 `prd.json` 和 `progress.txt` 复制进去
   - 用新的表头重置 `progress.txt`

**运行 `ralph.sh` 时，脚本会自动处理这件事**；但如果你是在多次运行之间手动更新 `prd.json`，请先完成归档。

---

## 保存前检查清单

写入 `prd.json` 之前，请确认：

- [ ] **上一次运行已归档**（如果已有 prd.json 且 branchName 不同，先归档）
- [ ] 每条 story 都足够小，能在一轮迭代中完成
- [ ] Stories 已按依赖顺序排列（schema -> backend -> UI）
- [ ] 每条 story 的标准里都有 `"Typecheck passes"`
- [ ] UI story 的标准里都有 `"Verify in browser using dev-browser skill"`
- [ ] 验收标准都可验证，不是模糊表述
- [ ] 没有任何 story 依赖后面的 story
