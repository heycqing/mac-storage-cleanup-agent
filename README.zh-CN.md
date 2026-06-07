# Mac Storage Cleanup Agent

一个通用的 macOS 存储清理 AI Agent 工作流，适用于 Codex、Claude Code、DeepSeek，以及其他能操作终端的 AI 编程助手。

它的核心原则不是“让 AI 直接删除文件”，而是：

```text
先只读盘点 -> 再风险分类 -> 等用户确认 -> 再执行清理 -> 最后复查空间
```

## 它能做什么

- 先用只读命令检查 macOS 磁盘空间。
- 找出大目录、缓存、构建产物、日志、包管理器缓存、应用残留。
- 把候选项分成低风险、中风险、高风险、禁止直接删除。
- 删除前必须等待用户确认。
- 清理后输出释放空间报告。

## 推荐开源目录结构

```text
mac-storage-cleanup-agent/
├── README.md          # 英文说明
├── README.zh-CN.md    # 中文说明
├── PROMPT.md          # 通用提示语，适合 DeepSeek 等任意 AI
├── SKILL.md           # Codex skill 适配
├── CLAUDE.md          # Claude Code 适配
└── LICENSE
```

## 在 Codex 中使用

把这个目录复制到 Codex skills 目录：

```bash
mkdir -p ~/.codex/skills
cp -R mac-storage-cleanup-agent ~/.codex/skills/
```

最终路径应该类似：

```text
~/.codex/skills/mac-storage-cleanup-agent/SKILL.md
```

然后重启 Codex，或新开一个 Codex 线程。

你可以这样说：

```text
使用 mac-storage-cleanup-agent skill 帮我清理 Mac 空间。
先只读盘点，不要删除任何文件，等我确认后再执行。
```

## 在 Claude Code 中使用

把 `CLAUDE.md` 放到项目或工作目录里，或者直接把其中内容复制给 Claude Code。

然后对 Claude Code 说：

```text
按照 CLAUDE.md 里的 Mac Storage Cleanup Agent 流程帮我清理 macOS 存储。
先只读盘点，不要删除任何内容，等我确认清理计划后再执行。
```

## 在 DeepSeek 或其他 AI 中使用

直接复制 `PROMPT.md` 的内容给 AI。

也可以这样开始：

```text
请按照下面的 macOS 存储清理流程工作：先只读盘点，列出候选清理项并分类风险；不要删除任何文件；等我确认后再执行低风险清理；最后复查释放空间。
```

## 安全模型

这个工作流故意设计得比较保守：

- 删除前必须先做只读盘点。
- 没有明确确认，不执行删除。
- 默认跳过用户文档、照片、视频、源码、数据库、密钥、证书和配置文件。
- 不确定用途的路径一律视为高风险。
- 尽量使用工具自带的清理命令，而不是直接递归删除。
- 清理结束后必须复查磁盘空间。

## 风险分类

### 低风险

通常可以清理，但仍然需要用户确认：

- 系统或应用缓存
- 临时日志
- 包管理器缓存
- 可重新生成的构建产物
- Xcode DerivedData
- 旧的临时文件

### 中风险

只给建议，不默认删除：

- 下载目录里的安装包
- 压缩包
- 重复文件
- 旧项目的构建目录
- 不再使用的软件安装包

### 高风险

默认跳过：

- 源码
- 文档
- 照片和视频
- 数据库
- 应用数据目录
- 不清楚用途的大文件

### 禁止自动删除

除非用户明确点名，否则不要碰：

- 密钥
- 证书
- 账号资料
- 配置文件
- SSH/GPG 相关文件
- 用户创作内容
- 未确认归属的数据

## 通用提示语

```text
你是我的 macOS 存储清理助手。请先只读盘点，不要删除任何文件。统计磁盘空间和大目录，列出候选清理项，并按低风险、中风险、高风险、禁止直接删除分类。给出路径、大小、类型、风险、建议动作和预计收益。只有在我明确确认后，才生成并执行低风险清理命令。清理后复查释放空间，并输出清理账单。
```

## 发布到 GitHub

建议仓库名：

```text
mac-storage-cleanup-agent
```

建议仓库描述：

```text
一个通用的 macOS 存储清理 AI Agent 工作流：先盘点、再分类、确认后清理、最后复查空间。
```

建议 topics：

```text
ai-agent
codex
claude-code
deepseek
macos
storage-cleanup
developer-tools
prompt-engineering
```

## 许可证

本项目使用 [MIT License](LICENSE)，允许他人自由使用、修改和分享这个工作流。

