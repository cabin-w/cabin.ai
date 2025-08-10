[English](#english) | [中文](#中文)

## 中文

### Cabin.AI — 轻量的流式 AI Chat App（Vue 3 + Vite + Element Plus）

> 在线演示：[`ai.cabin.ink`](https://ai.cabin.ink)

一个开箱即用的 Web 端 AI Chat 应用：无需后端，API Key 本地保存，支持流式输出、会话管理、导出/导入、消息级操作、暗黑模式与移动端适配。

> 完整源码已推送至 `code` 分支
#### 功能特性
- 流式输出（SSE）与超时保护：逐字呈现回答，超长文本不再卡死，可随时中止
- 对话管理：创建/切换/删除会话，持久化存储于 `localStorage`
- 设置面板：本地保存 API Key、模型名、角色设定与我的头像
- 消息级操作：复制整条消息、对最后一条助手消息“重新生成”、删除消息（用户消息不显示操作）
- Markdown + 代码高亮：内置 MarkdownIt 与 highlight.js，代码块一键复制
- 导出/导入当前对话（JSON）与一键清空
- 主题与自适应：深色/浅色主题切换，适配手机端抽屉侧栏和设置弹窗
- “更多”入口：导出/导入/清空操作统一收纳

#### 技术栈
- Vue 3 + Vite
- Element Plus（UI 组件）
- @vueuse/core（暗色模式切换等）
- MarkdownIt + highlight.js（Markdown 与代码高亮）
- 模型推理服务：SiliconFlow（兼容 OpenAI 风格接口）

#### 目录结构
```
src/
  App.vue                # 布局、侧栏、设置弹窗、移动端适配
  components/
    AIChat.vue           # 聊天主组件：流式输出、消息渲染与操作
```

#### 快速开始
环境要求：Node.js 18+（推荐 20+）

1) 安装依赖
```
npm install
# 或 pnpm i / yarn
```

2) 启动开发服务器
```
npm run dev
```

3) 构建与预览
```
npm run build
npm run preview
```

#### 使用说明
1) 设置：左侧“配置设置” → API Key / 模型名称 / 角色设定 / 我的头像
2) 发送消息：支持回车发送；生成时可“停止生成”
3) 消息操作：复制/重新生成（最后一条助手）/删除
4) 对话管理：“更多”可导出/导入/清空当前对话

#### 配置与定制
- 接口：`https://api.siliconflow.cn/v1/chat/completions`（兼容 OpenAI 格式）
- 流式：Fetch + ReadableStream 解析 `data:` 行，支持 `[DONE]`
- 主题：`useDark()` 管理暗色模式，样式可在 `AIChat.vue` 调整
- 移动端：侧栏抽屉化；设置弹窗在移动端宽度 ~92vw

#### 安全与隐私
- API Key 仅存本地 `localStorage`，前端直连服务商接口，请自行评估风险并遵循策略

#### 许可
- MIT

---

## English

### Cabin.AI — Lightweight streaming AI chat app (Vue 3 + Vite + Element Plus)

> Live Demo: [`ai.cabin.ink`](https://ai.cabin.ink) 

A ready-to-use web AI chat app: no backend required. Your API key is stored locally. It supports streaming responses, conversation management, import/export, per-message actions, dark mode, and mobile-friendly UI.

> Full source code is in the `code` branch.
#### Features
- Streaming (SSE) with timeout protection: token-by-token rendering; stop anytime
- Conversation management: create/switch/delete; persisted in `localStorage`
- Settings panel: API key, model name, system prompt, and my avatar
- Per-message actions: copy message, “regenerate” last assistant message, delete (no actions on user messages)
- Markdown + code highlighting with one-click copy
- Export/import current conversation (JSON) and clear with one click
- Theming & responsiveness: dark/light mode; mobile drawer sidebar and responsive settings dialog
- “More” menu to keep the UI clean

#### Tech Stack
- Vue 3 + Vite
- Element Plus (UI)
- @vueuse/core (dark mode, etc.)
- MarkdownIt + highlight.js
- Inference via SiliconFlow (OpenAI-compatible API)

#### Project Structure
```
src/
  App.vue                # Layout, sidebar, settings dialog, mobile adaption
  components/
    AIChat.vue           # Core chat component: streaming, render, actions
```

#### Getting Started
Requirements: Node.js 18+ (20+ recommended)

1) Install
```
npm install
```

2) Dev server
```
npm run dev
```

3) Build & preview
```
npm run build
npm run preview
```

#### Usage
1) Open Settings: API key / model / system prompt / my avatar
2) Send messages; streaming output; you can stop generation
3) Message actions: copy / regenerate last assistant / delete
4) Manage conversations: export/import/clear via “More”

#### Configuration
- Endpoint: `https://api.siliconflow.cn/v1/chat/completions` (OpenAI-compatible)
- Streaming: parse `data:` lines from ReadableStream; supports `[DONE]`
- Theming: dark mode via `useDark()`; tweak styles in `AIChat.vue`
- Mobile: drawer sidebar; settings dialog ~92vw on phones

#### Security & Privacy
- API key lives in browser `localStorage` only. You connect directly to provider endpoints. Evaluate the risk and comply with provider policies.

#### License
- MIT
