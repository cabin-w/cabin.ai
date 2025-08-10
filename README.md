## Cabin.AI — 轻量本地运行的流式aichatbot（Vue 3 + Vite + Element Plus）

> 在线演示：[`ai.cabin.ink`](https://ai.cabin.ink)

一个开箱即用的 Web 端 AI Chat 应用：无需后端，API Key 本地保存，支持流式输出、会话管理、导出/导入、消息级操作、暗黑模式与移动端适配。

### 功能特性
- 流式输出（SSE）与超时保护：逐字呈现回答，超长文本不再卡死，可随时中止
- 对话管理：创建/切换/删除会话，持久化存储于 `localStorage`
- 设置面板：本地保存 API Key、模型名、角色设定与我的头像
- 消息级操作：复制整条消息、对最后一条助手消息“重新生成”、删除消息
- Markdown + 代码高亮：内置 MarkdownIt 与 highlight.js，代码块一键复制
- 导出/导入当前对话（JSON）与一键清空
- 主题与自适应：深色/浅色主题切换，适配手机端抽屉侧栏和设置弹窗
- “更多”入口：导出/导入/清空操作统一收纳，界面更简洁


### 技术栈
- Vue 3 + Vite
- Element Plus（UI 组件）
- @vueuse/core（暗色模式切换等）
- MarkdownIt + highlight.js（Markdown 与代码高亮）

### 目录结构（关键部分）
```
src/
  App.vue                # 布局、侧栏、设置弹窗、移动端适配
  components/
    AIChat.vue           # 聊天主组件：流式输出、消息渲染与操作
```

### 快速开始
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

3) 构建生产包
```
npm run build
npm run preview
```

### 使用说明
1) 打开应用后，点击左侧“配置设置”进入设置弹窗：
   - API Key：填写你的服务商 Key（本项目默认使用 SiliconFlow 的兼容接口）
   - 模型名称：例如 `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`
   - 角色设定：对话的系统提示词（可为空）
   - 设置头像：
     - 若填写图片 URL（含 `http(s)` 或 `data:image`），头像将显示该图片
     - 若填写文本（1-2 字），将以文字头像显示
     - 留空时使用默认头像地址（见上）

2) 在主界面输入消息后回车/点击“发送”，可看到流式输出；生成过程中可“停止生成”。

3) 消息操作：
   - 复制（任意消息）
   - 重新生成（仅最后一条助手消息且非生成中）
   - 删除（任意消息）

4) 对话管理：
   - 左侧会话列表可创建/切换/删除对话，内容持久化至 `localStorage`
   - “更多”按钮：导出当前对话 JSON、导入 JSON、清空当前对话

### 配置与定制
- 接口地址：使用 `AIChat.vue` 中的 `https://api.siliconflow.cn/v1/chat/completions`（兼容 OpenAI 格式）
- 流式实现：Fetch + ReadableStream 解析 `data:` 行（SSE），支持 `[DONE]` 结束标记
- 主题与样式：
  - 深色模式通过 `@vueuse/core` 的 `useDark()` 自动持久化
  - Markdown 与代码高亮样式在 `AIChat.vue` 中可按需调整
- 移动端适配：
  - 侧栏在移动端抽屉化，主内容左上提供开合按钮且适配暗/亮主题
  - 设置弹窗移动端自动放宽至 `~92vw`，行距/边距更紧凑

### 安全与隐私
- API Key 仅保存在浏览器本地 `localStorage`，前端直连服务商接口；请自行评估并遵循服务商的使用策略




### 许可与致谢
- License：MIT（根据你项目需要可调整）
- UI 与生态：Element Plus、@vueuse/core
- Markdown/高亮：MarkdownIt、highlight.js
- 模型推理服务：SiliconFlow（兼容 OpenAI 风格接口）

