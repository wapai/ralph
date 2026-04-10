# Ralph 代理说明

## 概览

Ralph 是一个自主 AI 代理循环，会反复运行 AI 编码工具（Amp、Claude Code 或 Codex），直到所有 PRD 条目都完成。每一轮都是一个全新的实例，拥有干净的上下文。

## 命令

```bash
# 启动 flowchart 开发服务器
cd flowchart && npm run dev

# 构建 flowchart
cd flowchart && npm run build

# 使用 Amp 运行 Ralph（默认）
./ralph.sh [max_iterations]

# 使用 Claude Code 运行 Ralph
./ralph.sh --tool claude [max_iterations]

# 使用 Codex 运行 Ralph
./ralph.sh --tool codex [max_iterations]
```

## 关键文件

- `ralph.sh` - 启动全新 AI 实例的 bash 循环（支持 `--tool amp`、`--tool claude` 或 `--tool codex`）
- `prompt.md` - 提供给每个 Amp 实例的说明
- `CLAUDE.md` - 提供给每个 Claude Code 实例的说明
- `CODEX.md` - 提供给每个 Codex 实例的说明
- `prd.json.example` - PRD 格式示例
- `flowchart/` - 用于说明 Ralph 工作方式的交互式 React Flow 图

## 流程图

`flowchart/` 目录中包含一个基于 React Flow 的交互式可视化。它适合用于演示，你可以逐步点击查看每个步骤及其动画。

本地运行方式：
```bash
cd flowchart
npm install
npm run dev
```

## 模式

- 每一轮都会启动一个拥有干净上下文的全新 AI 实例（Amp、Claude Code 或 Codex）
- 记忆通过 git 历史、`progress.txt` 和 `prd.json` 持续保留
- Story 必须足够小，能在一个上下文窗口内完成
- 发现可复用模式后，要始终更新 `AGENTS.md`，供后续迭代使用
