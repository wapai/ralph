# Ralph

![Ralph](ralph.webp)

Ralph 是一个自主 AI 代理循环，会反复运行 AI 编码工具（[Amp](https://ampcode.com)、[Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或 Codex CLI），直到所有 PRD 条目都完成。每一轮都是一个全新的实例，拥有干净的上下文。记忆通过 git 历史、`progress.txt` 和 `prd.json` 保留下来。

它基于 [Geoffrey Huntley 提出的 Ralph 模式](https://ghuntley.com/ralph/)。

[这里是作者关于如何使用 Ralph 的长文](https://x.com/ryancarson/status/2008548371712135632)

## 前置条件

- 安装并登录以下任意一种 AI 编码工具：
  - [Amp CLI](https://ampcode.com)（默认）
  - [Claude Code](https://docs.anthropic.com/en/docs/claude-code)（`npm install -g @anthropic-ai/claude-code`）
  - Codex CLI（先执行 `codex`，再执行 `codex login`）
- 安装 `jq`（macOS 可执行 `brew install jq`）
- 你的项目本身是一个 git 仓库

## 安装方式

### 方式 1：复制到你的项目

把 Ralph 相关文件复制到你的项目中：

```bash
# 在项目根目录执行
mkdir -p scripts/ralph
cp /path/to/ralph/ralph.sh scripts/ralph/

# 复制你要使用的 AI 工具对应的提示模板：
cp /path/to/ralph/prompt.md scripts/ralph/prompt.md    # 用于 Amp
# 或
cp /path/to/ralph/CLAUDE.md scripts/ralph/CLAUDE.md    # 用于 Claude Code
# 或
cp /path/to/ralph/CODEX.md scripts/ralph/CODEX.md      # 用于 Codex

chmod +x scripts/ralph/ralph.sh
```

### 方式 2：全局安装 skills（Amp）

把 skills 复制到 Amp 或 Claude 的配置目录中，以便在所有项目里复用。

用于 AMP：
```bash
cp -r skills/prd ~/.config/amp/skills/
cp -r skills/ralph ~/.config/amp/skills/
```

用于 Claude Code（手动）：
```bash
cp -r skills/prd ~/.claude/skills/
cp -r skills/ralph ~/.claude/skills/
```

### 方式 3：作为 Claude Code Marketplace 使用

把 Ralph marketplace 添加到 Claude Code：

```bash
/plugin marketplace add snarktank/ralph
```

然后安装 skills：

```bash
/plugin install ralph-skills@ralph-marketplace
```

安装后可用的 skills：
- `/prd`：生成产品需求文档
- `/ralph`：把 PRD 转成 `prd.json` 格式

当你这样对 Claude 提问时，这些 skill 会被自动触发：
- “create a prd”、“write prd for”、“plan this feature”
- “convert this prd”、“turn into ralph format”、“create prd.json”

### 配置 Amp 自动交接（推荐）

把下面这段加到 `~/.config/amp/settings.json`：

```json
{
  "amp.experimental.autoHandoff": { "context": 90 }
}
```

这样在上下文快满时会自动交接，让 Ralph 能处理超出单个上下文窗口的大任务。

## 工作流程

### 1. 创建 PRD

使用 PRD skill 生成详细的需求文档：

```
Load the prd skill and create a PRD for [your feature description]
```

回答澄清问题后，skill 会把输出保存到 `tasks/prd-[feature-name].md`。

### 2. 转换 PRD 为 Ralph 格式

使用 Ralph skill 把 markdown PRD 转换成 JSON：

```
Load the ralph skill and convert tasks/prd-[feature-name].md to prd.json
```

这样会生成 `prd.json`，其中的 user stories 会按适合自主执行的结构组织好。

### 3. 运行 Ralph

```bash
# 使用 Amp（默认）
./scripts/ralph/ralph.sh [max_iterations]

# 使用 Claude Code
./scripts/ralph/ralph.sh --tool claude [max_iterations]

# 使用 Codex
./scripts/ralph/ralph.sh --tool codex [max_iterations]
```

默认迭代次数是 10。你可以通过 `--tool amp`、`--tool claude` 或 `--tool codex` 选择 AI 编码工具。

Ralph 会执行以下流程：
1. 根据 PRD 里的 `branchName` 创建功能分支
2. 找到优先级最高且 `passes: false` 的 story
3. 实现这一个 story
4. 运行质量检查（typecheck、tests）
5. 如果检查通过则提交代码
6. 更新 `prd.json`，把该 story 标记为 `passes: true`
7. 把经验和进展追加到 `progress.txt`
8. 重复以上过程，直到所有 story 都通过，或达到最大迭代次数

## 关键文件

| 文件 | 用途 |
|------|------|
| `ralph.sh` | 启动全新 AI 实例的 bash 循环（支持 `--tool amp`、`--tool claude` 或 `--tool codex`） |
| `prompt.md` | Amp 使用的提示模板 |
| `CLAUDE.md` | Claude Code 使用的提示模板 |
| `CODEX.md` | Codex 使用的提示模板 |
| `prd.json` | 带有 `passes` 状态的 user stories 任务列表 |
| `prd.json.example` | PRD 格式参考示例 |
| `progress.txt` | 仅追加的经验与进展日志 |
| `skills/prd/` | 用于生成 PRD 的 skill（适用于 Amp 和 Claude Code） |
| `skills/ralph/` | 用于把 PRD 转成 JSON 的 skill（适用于 Amp 和 Claude Code） |
| `.claude-plugin/` | 用于 Claude Code marketplace 发现的插件清单 |
| `flowchart/` | Ralph 工作流程的交互式可视化 |

## 流程图

[![Ralph Flowchart](ralph-flowchart.png)](https://snarktank.github.io/ralph/)

**[查看交互式流程图](https://snarktank.github.io/ralph/)**：点击查看每个步骤及其动画。

`flowchart/` 目录中包含其源码。本地运行方式如下：

```bash
cd flowchart
npm install
npm run dev
```

## 核心概念

### 每一轮迭代 = 全新上下文

每一轮都会启动一个 **全新的 AI 实例**（Amp、Claude Code 或 Codex），拥有干净上下文。轮次之间能保留的记忆只有：
- git 历史（前几轮的提交）
- `progress.txt`（经验和上下文）
- `prd.json`（哪些 story 已经完成）

### 小任务

每一条 PRD 项目都应该足够小，能在一个上下文窗口内完成。如果任务太大，LLM 会在完成前耗尽上下文，最后生成质量很差的代码。

合适大小的 story：
- 添加一个数据库字段和 migration
- 在现有页面里增加一个 UI 组件
- 修改一个 server action 的逻辑
- 给列表增加一个筛选下拉框

太大的任务（应继续拆分）：
- “构建整个 dashboard”
- “添加认证系统”
- “重构整个 API”

### AGENTS.md 更新非常关键

每一轮结束后，Ralph 都会把经验写入相关的 `AGENTS.md` 文件。这非常重要，因为 AI 编码工具会自动读取这些文件，所以后续迭代以及未来的人类开发者都能受益于这些模式、坑点和约定。

适合写进 `AGENTS.md` 的内容示例：
- 发现的模式（“这个代码库在 Y 场景下用 X”）
- 容易踩的坑（“改 W 时别忘了同步更新 Z”）
- 有用的上下文（“settings panel 在组件 X 里”）

### 反馈回路

Ralph 只有在存在反馈回路时才真正有效：
- Typecheck 能抓出类型错误
- Tests 能验证行为是否正确
- CI 必须保持绿色，否则错误会在多轮迭代中累积

### UI Story 的浏览器验证

前端相关 story 的验收条件里必须包含 “Verify in browser using dev-browser skill”。Ralph 会使用 dev-browser skill 打开页面、与 UI 交互，并确认改动确实有效。

### 停止条件

当所有 story 都变成 `passes: true` 时，Ralph 会输出 `<promise>COMPLETE</promise>`，然后退出循环。

## 调试

查看当前状态：

```bash
# 查看哪些 story 已完成
cat prd.json | jq '.userStories[] | {id, title, passes}'

# 查看之前迭代积累的经验
cat progress.txt

# 查看 git 历史
git log --oneline -10
```

## 自定义提示模板

把 `prompt.md`（Amp）、`CLAUDE.md`（Claude Code）或 `CODEX.md`（Codex）复制到你的项目后，你应该按项目实际情况进行定制：
- 加入项目自己的质量检查命令
- 补充代码库约定
- 记录这个技术栈里的常见坑

## 归档

当你开始一个新功能（即 `branchName` 不同）时，Ralph 会自动归档之前的运行结果。归档目录格式为 `archive/YYYY-MM-DD-feature-name/`。

## 参考资料

- [Geoffrey Huntley 关于 Ralph 的文章](https://ghuntley.com/ralph/)
- [Amp 文档](https://ampcode.com/manual)
- [Claude Code 文档](https://docs.anthropic.com/en/docs/claude-code)
