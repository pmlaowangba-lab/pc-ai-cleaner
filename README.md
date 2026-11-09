# pc-ai-cleaner

Windows 和 macOS 都能用的 AI 环境管家 Skill。Agent 按这份说明做三件事：找出磁盘空间被谁占了、清掉可再生的 AI 缓存、判断这台机器能舒服地跑多大的本地模型。

扫描默认只读。没有逐条确认，不会删除或移动任何文件。真要清的时候，先搬到隔离目录并写清单，而不是直接 `rm`。

## 它做什么

- **磁盘分级**：用户目录、临时文件、浏览器缓存、开发缓存，按占用从大到小分成可放心删、按需清理、核心依赖别碰。
- **AI 缓存专项**：Ollama、HuggingFace、ComfyUI、uv、pip、conda、npm、pnpm、Homebrew，以及 Codex / Claude / Cursor 一类 Agent 的日志和会话轨迹。模型权重用工具自己的卸载命令处理，不手删 blob。
- **硬件与环境**：Windows 看显存和 CUDA；Mac 看 Apple 芯片统一内存和 Metal。顺便查多个 Python、装错解释器、PATH 冲突。

## 它不做什么

- 不改注册表、启动项、SIP、`nvram`。
- 不代删桌面、文档、下载、照片图库里的个人文件，只报告。
- 不安装清理软件，也不在笼统的一句「全清」之后动手。

## 安装

任意支持 Agent Skills 的工具里，让 Agent 安装这个仓库：

```text
帮我安装这个 skill：https://github.com/pmlaowangba-lab/pc-ai-cleaner
```

也可以把本仓库放到对应工具的 skills 目录，目录名用 `pc-ai-cleaner`，保证其中有 `SKILL.md`。

## 怎么触发

直接说人话即可，例如：

```text
C 盘满了
Macintosh HD 满了
看看 Ollama 和 HuggingFace 占了多少
这台机器能跑多大的模型
只读体检一下电脑，先别删
```

## 平台

| 系统 | 状态 |
|------|------|
| Windows 10 / 11，PowerShell 5.1 或 7 | 规则按本机清理场景写过，并用只读命令核对过 |
| macOS（Apple 芯片与 Intel） | 同一套分级和命令约定；缓存真实位置以 `uv cache dir`、`pip cache dir`、`brew --cache` 等工具自报路径为准 |

Linux、容器、云主机不在范围内。如果当前环境和用户要查的那台电脑不是同一台，Skill 会停下来，改交付只读脚本。

## 许可

[MIT](LICENSE)
