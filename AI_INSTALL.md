# AI Agent Setup / AI 安装说明

This file is designed to be pasted into Codex, Claude Code, Gemini CLI, Cursor, or another coding agent.

本文件可直接交给 Codex、Claude Code、Gemini CLI、Cursor 或其他编程智能体执行。

## Ready-to-paste prompt / 可直接复制的提示词

```text
Install SecTest Atlas from https://github.com/cndoin/sectest-atlas.git into a new local directory named sectest-atlas. Do not install third-party packages because this is a dependency-free static site. Verify that index.html, docs/handbook.md, assets/cover.svg, and assets/interface-preview.svg exist. Start a local static server on an available loopback port (prefer 8080), open the site, and confirm that the title, sidebar search, theme toggle, checklist persistence, mobile layout, and internal links work. Do not run any security-testing commands from the handbook and do not test external targets.

请把 https://github.com/cndoin/sectest-atlas.git 安装到名为 sectest-atlas 的新本地目录。它是无依赖静态站点，不要安装第三方软件包。确认 index.html、docs/handbook.md、assets/cover.svg、assets/interface-preview.svg 均存在；在可用的本机回环端口启动静态服务器（优先 8080），打开页面并验证标题、侧栏搜索、主题切换、检查项进度保存、移动端布局和内部链接。不要执行手册里的任何安全测试命令，也不要测试外部目标。
```

## Minimal agent commands / 最小命令集

```bash
git clone https://github.com/cndoin/sectest-atlas.git
cd sectest-atlas
git status --short --branch
python -m http.server 8080
```

## Verification contract / 验收约定

The agent should report:

1. Installation path and checked-out commit.
2. Local preview URL.
3. Whether every required file exists.
4. Whether the HTML script parses and all `#anchor` links resolve.
5. Any browser-console error, broken asset, or layout problem.

AI 应返回：安装路径与提交版本、本地预览地址、必要文件检查、锚点与脚本检查结果，以及任何控制台错误、资源丢失或布局问题。
