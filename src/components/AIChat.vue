<script setup>
import { ref, watch, onMounted, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import 'highlight.js/styles/atom-one-light.css' // 使用浅色主题

const md = new MarkdownIt({
  html: true,
  breaks: true,
  linkify: true,
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        const highlighted = hljs.highlight(str, { language: lang, ignoreIllegals: true }).value
        return `<pre class="hljs ${lang}"><button class="code-copy-btn">复制代码</button><code>${highlighted}</code></pre>`
      } catch (__) {}
    }
    return `<pre class="hljs"><button class="code-copy-btn">复制代码</button><code>${md.utils.escapeHtml(str)}</code></pre>`
  }
})

const props = defineProps({
  apiKey: {
    type: String,
    required: true
  },
  promptText: {
    type: String,
    default: ''
  },
  chatId: {
    type: Number,
    required: true
  },
  initialMessages: {
    type: Array,
    default: () => []
  },
  modelName: {
    type: String,
    default: 'deepseek-ai/DeepSeek-R1-Distill-Qwen-32B'
  },
  userAvatar: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['update-messages'])

const inputMessage = ref('')
const chatMessages = ref([])
const loading = ref(false)
const messagesContainer = ref(null)
const fileInputRef = ref(null)
const currentController = ref(null)
const abortedByUser = ref(false)

const scrollToBottom = async () => {
  await nextTick()
  try {
    const container = messagesContainer.value
    if (container) {
      container.scrollTop = container.scrollHeight
    }
  } catch (_) {}
}

// 监听聊天ID变化，加载对应的消息
watch(() => props.chatId, async () => {
  chatMessages.value = [...props.initialMessages]
  await scrollToBottom()
}, { immediate: true })

// 监听消息变化，通知父组件更新
watch(chatMessages, (newMessages) => {
  emit('update-messages', newMessages)
  scrollToBottom()
}, { deep: true })

const sendMessage = async () => {
  if (!inputMessage.value.trim()) {
    ElMessage.warning('请输入消息')
    return
  }

  if (!props.apiKey) {
    ElMessage.warning('请先设置 API Key')
    return
  }

  const userMessage = inputMessage.value
  chatMessages.value.push({
    role: 'user',
    content: userMessage
  })
  
  loading.value = true
  inputMessage.value = ''

  // 发送给接口的历史（包含刚添加的 user），系统提示词使用 system 角色
  const messagesForAPI = [
    {
      role: 'system',
      content: props.promptText || '你扮演角色:cabin'
    },
    ...chatMessages.value
  ]

  // 先插入一个占位的助手消息，用于流式追加
  chatMessages.value.push({ role: 'assistant', content: '' })
  const assistantIndex = chatMessages.value.length - 1

  const options = {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${props.apiKey}`,
      'Content-Type': 'application/json',
      Accept: 'text/event-stream'
    },
    body: JSON.stringify({
      model: props.modelName,
      stream: true,
      messages: messagesForAPI
    })
  }

  try {
    const controller = new AbortController()
    currentController.value = controller
    // 更长的总体超时（2 分钟）
    const timeoutId = setTimeout(() => controller.abort(), 120000)

    const response = await fetch('https://api.siliconflow.cn/v1/chat/completions', {
      ...options,
      signal: controller.signal
    })

    clearTimeout(timeoutId)

    if (!response.ok) {
      let errText = ''
      try { errText = await response.text() } catch (_) {}
      throw new Error(`HTTP ${response.status}: ${errText || '请求失败'}`)
    }

    // 优先尝试按 SSE 流式解析
    const reader = response.body?.getReader()
    const decoder = new TextDecoder('utf-8')
    if (reader) {
      let buffer = ''
      while (true) {
        const { value, done } = await reader.read()
        if (done) break
        buffer += decoder.decode(value, { stream: true })
        const lines = buffer.split(/\n/)
        buffer = lines.pop() || ''
        for (const line of lines) {
          const trimmed = line.trim()
          if (!trimmed) continue
          if (trimmed.startsWith('data:')) {
            const dataStr = trimmed.replace(/^data:\s*/, '')
            if (dataStr === '[DONE]') {
              break
            }
            try {
              const json = JSON.parse(dataStr)
              const choice = json.choices?.[0]
              const delta = choice?.delta?.content ?? choice?.message?.content ?? ''
              if (delta) {
                chatMessages.value[assistantIndex].content += delta
              }
            } catch (_) {
              // 忽略无法解析的分片
            }
          }
        }
        // 每批次滚动到底部
        scrollToBottom()
      }
    } else {
      // 非流式兜底
      const data = await response.json()
      const content = data?.choices?.[0]?.message?.content || ''
      chatMessages.value[assistantIndex].content = content
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      if (abortedByUser.value) {
        ElMessage.info('已停止生成')
      } else {
        ElMessage.error('请求超时，请稍后重试')
      }
    } else {
      ElMessage.error('发送消息失败')
    }
    // 如果占位还为空，移除它
    const msg = chatMessages.value[assistantIndex]
    if (msg && msg.role === 'assistant' && !msg.content) {
      chatMessages.value.splice(assistantIndex, 1)
    }
    console.error(error)
  } finally {
    loading.value = false
    currentController.value = null
    abortedByUser.value = false
  }
}

const stopGeneration = () => {
  if (currentController.value) {
    abortedByUser.value = true
    try { currentController.value.abort() } catch (_) {}
  }
}

const retryLast = async () => {
  if (loading.value) return
  if (chatMessages.value.length === 0) return
  // 如果最后一条是助手，先移除
  if (chatMessages.value[chatMessages.value.length - 1]?.role === 'assistant') {
    chatMessages.value.pop()
  }
  // 取最后一条用户消息并移除
  let lastUserIndex = -1
  for (let i = chatMessages.value.length - 1; i >= 0; i--) {
    if (chatMessages.value[i].role === 'user') { lastUserIndex = i; break }
  }
  if (lastUserIndex === -1) return
  const userText = chatMessages.value[lastUserIndex].content
  chatMessages.value.splice(lastUserIndex, 1)
  // 直接复用发送逻辑，但不依赖输入框
  inputMessage.value = userText
  await nextTick()
  sendMessage()
}

// 导出 / 导入 / 清空 / 复制消息
const exportCurrentChat = () => {
  try {
    const payload = {
      chatId: props.chatId,
      model: props.modelName,
      systemPrompt: props.promptText || '你扮演角色:cabin',
      messages: chatMessages.value,
      exportedAt: new Date().toISOString()
    }
    const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `chat-${props.chatId || 'session'}.json`
    a.click()
    URL.revokeObjectURL(url)
  } catch (e) {
    ElMessage.error('导出失败')
  }
}

const openImportDialog = () => {
  try { fileInputRef.value && fileInputRef.value.click() } catch (_) {}
}

const onImportFileChange = async (event) => {
  const file = event?.target?.files?.[0]
  if (!file) return
  try {
    const text = await file.text()
    const data = JSON.parse(text)
    const importedMessages = Array.isArray(data)
      ? data
      : Array.isArray(data?.messages) ? data.messages : []
    if (!Array.isArray(importedMessages) || importedMessages.length === 0) {
      throw new Error('数据格式不正确')
    }
    const sanitized = importedMessages
      .filter(m => m && typeof m.content === 'string' && (m.role === 'user' || m.role === 'assistant' || m.role === 'system'))
      .map(m => ({ role: m.role, content: m.content }))
    if (sanitized.length === 0) throw new Error('没有有效的消息')
    chatMessages.value = sanitized
    ElMessage.success('导入成功')
  } catch (e) {
    console.error(e)
    ElMessage.error('导入失败：JSON 格式不正确或缺少 messages')
  } finally {
    // 允许重复选择同一个文件
    if (event?.target) event.target.value = ''
  }
}

const clearCurrentChat = async () => {
  try {
    await ElMessageBox.confirm('确定要清空当前对话吗？该操作不可撤销。', '清空对话', {
      type: 'warning',
      confirmButtonText: '清空',
      cancelButtonText: '取消'
    })
    chatMessages.value = []
    ElMessage.success('已清空当前对话')
  } catch (_) {
    // 用户取消
  }
}

const copyMessage = async (message) => {
  try {
    await navigator.clipboard.writeText(message?.content || '')
    ElMessage.success('消息已复制')
  } catch (_) {
    ElMessage.error('复制失败')
  }
}

// 当 API Key 或提示词变化时，清空对话
watch([() => props.apiKey, () => props.promptText], () => {
  chatMessages.value = []
})

// 添加渲染 Markdown 的方法
const renderMessage = (content) => {
  return md.render(content)
}

// 添加复制功能
const copyCode = (event) => {
  const codeBlock = event.target.parentElement.querySelector('code')
  if (codeBlock) {
    navigator.clipboard.writeText(codeBlock.textContent)
    ElMessage.success('代码已复制')
  }
}

onMounted(() => {
  // 添加错误处理
  try {
    chatMessages.value = [...props.initialMessages]
    console.log('AIChat component mounted successfully')
    scrollToBottom()
  } catch (error) {
    console.error('Error initializing AIChat:', error)
  }
})

const isImageUrl = (val) => {
  if (!val || typeof val !== 'string') return false
  const lower = val.trim().toLowerCase()
  if (lower.startsWith('http://') || lower.startsWith('https://') || lower.startsWith('data:image')) return true
  return (
    lower.endsWith('.png') ||
    lower.endsWith('.jpg') ||
    lower.endsWith('.jpeg') ||
    lower.endsWith('.gif') ||
    lower.endsWith('.svg') ||
    lower.endsWith('.webp')
  )
}

const deleteMessage = (index) => {
  if (loading.value) return
  if (index < 0 || index >= chatMessages.value.length) return
  chatMessages.value.splice(index, 1)
}
</script>

<template>
  <div class="chat-container">
    <div class="chat-messages" ref="messagesContainer">
      <div v-for="(message, index) in chatMessages" 
           :key="index" 
           :class="['message', message.role]">
        <div class="message-content">
          <div class="message-avatar">
            <el-avatar 
              :size="36" 
              :class="message.role"
            >
              <template v-if="message.role === 'user'">
                <template v-if="isImageUrl(props.userAvatar)">
                  <img :src="props.userAvatar" alt="user" style="width:100%;height:100%;object-fit:cover" />
                </template>
                <template v-else>
                  {{ props.userAvatar || '我' }}
                </template>
              </template>
              <template v-else>
                AI
              </template>
            </el-avatar>
          </div>
          <div class="message-bubble">
            <div class="message-content-wrapper">
              <div class="markdown-body" v-html="renderMessage(message.content)" @click="e => {
                if (e.target.classList.contains('code-copy-btn')) {
                  copyCode(e)
                }
              }"></div>
              <div v-if="message.role !== 'user'" class="message-actions-row">
                <el-button text circle class="icon-action-button" @click="copyMessage(message)">
                  <el-icon><DocumentCopy /></el-icon>
                </el-button>
                <el-button v-if="message.role === 'assistant' && index === chatMessages.length - 1 && !loading" text circle class="icon-action-button" @click="retryLast">
                  <el-icon><RefreshRight /></el-icon>
                </el-button>
                <el-button text circle class="icon-action-button" @click="deleteMessage(index)">
                  <el-icon><Delete /></el-icon>
                </el-button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <div class="input-container">
        <el-input
          class="message-input"
          v-model="inputMessage"
          type="textarea"
          :rows="3"
          placeholder="输入消息...(Enter 发送，Shift + Enter 换行)"
          :disabled="loading"
          resize="none"
          @keyup.enter.exact="sendMessage"
          @keyup.enter.shift.exact.prevent="inputMessage += '\n'"
        >
        </el-input>
        <div class="input-actions">
          <template v-if="!loading">
            <el-button 
              class="send-button"
              type="primary" 
              :loading="false"
              @click="sendMessage"
              :disabled="!inputMessage.trim()"
            >
              <el-icon><Position /></el-icon>
              <span>发送</span>
            </el-button>
          </template>
          <template v-else>
            <el-button 
              class="send-button stop-button"
              type="danger"
              plain
              @click="stopGeneration"
            >
              停止生成
            </el-button>
          </template>
          <el-dropdown class="more-actions">
            <span class="el-dropdown-link">
              <el-button class="send-button">更多</el-button>
            </span>
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item @click="exportCurrentChat">导出对话</el-dropdown-item>
                <el-dropdown-item @click="openImportDialog">导入对话</el-dropdown-item>
                <el-dropdown-item divided @click="clearCurrentChat">清空当前对话</el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
          <input type="file" ref="fileInputRef" accept="application/json" style="display:none" @change="onImportFileChange" />
        </div>
      </div>
    </div>
  </div>
</template>

<style>
/* 全局暗色主题样式 */
html.dark {
  --chat-bg-color: #1a1b1e;
  --chat-border-color: #2c2c30;
  --chat-bubble-bg: #2c2c30;
  --chat-bubble-user-bg: #313136;
  --chat-text-color: #e2e8f0;  /* 改为稍微暗一点的白色 */
  --chat-secondary-text: #94a3b8;
  --chat-code-bg: #1f1f23;
  --chat-code-border: #2c2c30;
}

html.dark .chat-container {
  background-color: var(--chat-bg-color);
}

html.dark .markdown-body {
  color: var(--chat-text-color);
}

html.dark .markdown-body p,
html.dark .markdown-body ul,
html.dark .markdown-body ol,
html.dark .markdown-body li {
  color: #cbd5e1;  /* 普通文本使用更柔和的颜色 */
}

html.dark .markdown-body strong {
  color: #818cf8;
}

html.dark .markdown-body em {
  color: #93c5fd;
}

html.dark .markdown-body h1,
html.dark .markdown-body h2,
html.dark .markdown-body h3,
html.dark .markdown-body h4,
html.dark .markdown-body h5,
html.dark .markdown-body h6 {
  color: #e2e8f0;  /* 标题使用稍亮一点的颜色 */
}

html.dark .markdown-body blockquote {
  background: var(--chat-bubble-user-bg);
  border-left-color: #818cf8;
  color: #93c5fd;
}

html.dark .markdown-body pre {
  background: var(--chat-code-bg) !important;
  border-color: var(--chat-code-border);
}

html.dark .markdown-body pre::before {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
}

html.dark .markdown-body pre code {
  color: #e5e7eb;
}

html.dark .markdown-body :not(pre) > code {
  background: var(--chat-bubble-user-bg);
  color: #818cf8;
}

html.dark .markdown-body a {
  color: #818cf8;
  border-bottom-color: rgba(129, 140, 248, 0.2);
}

html.dark .markdown-body a:hover {
  border-bottom-color: #818cf8;
}

html.dark .markdown-body table th {
  background: var(--chat-bubble-user-bg);
  color: #818cf8;
  border-color: var(--chat-border-color);
}

html.dark .markdown-body table td {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

/* 代码高亮暗色主题 */
html.dark .hljs {
  color: #e5e7eb;
}

html.dark .hljs-keyword,
html.dark .hljs-title.function_ {
  color: #818cf8;
}

html.dark .hljs-string {
  color: #7dd3fc;
}

html.dark .hljs-comment {
  color: #64748b;
}

html.dark .hljs-variable,
html.dark .hljs-attr {
  color: #93c5fd;
}

html.dark .hljs-number {
  color: #67e8f9;
}

html.dark .hljs-title.class_ {
  color: #818cf8;
}

html.dark .markdown-body pre .code-copy-btn {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-secondary-text);
}

html.dark .markdown-body pre .code-copy-btn:hover {
  background: var(--chat-bubble-user-bg);
  border-color: #818cf8;
  color: #818cf8;
}

/* 输入区域暗色主题 */
html.dark .input-area {
  background: var(--el-bg-color);
  border-color: var(--chat-border-color);
}

html.dark .input-tips {
  color: var(--chat-secondary-text);
}

/* 输入框暗色主题 */
html.dark .el-textarea__inner,
html.dark .el-textarea .el-textarea__inner {
  background: var(--chat-bubble-bg) !important;
  border-color: var(--chat-border-color) !important;
  color: var(--chat-text-color) !important;
}

html.dark .el-textarea__inner:hover,
html.dark .el-textarea .el-textarea__inner:hover {
  background: var(--chat-bubble-user-bg) !important;
  border-color: #818cf8 !important;
}

html.dark .el-textarea__inner:focus,
html.dark .el-textarea .el-textarea__inner:focus {
  background: var(--chat-bubble-user-bg) !important;
  border-color: #818cf8 !important;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.2) !important;
}

html.dark .send-button {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: #818cf8;
}

html.dark .send-button:not(:disabled):hover {
  background: #818cf8;
  color: white;
  border: none;
}

html.dark .send-button:disabled {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-secondary-text);
}

/* 滚动条暗色主题 */
html.dark .chat-messages::-webkit-scrollbar-thumb {
  background: #4b5563;
}

html.dark .chat-messages:hover::-webkit-scrollbar-thumb {
  background: #6b7280;
}
</style>

<style scoped>
.chat-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f8fafc;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.message {
  margin-bottom: 24px;
  display: flex;
  flex-direction: column;
}

.message.assistant {
  align-items: flex-start;
}

.message.user {
  align-items: flex-end;
}

.message-content {
  display: flex;
  gap: 12px;
  align-items: flex-start;
  max-width: 85%;
}

.message.user .message-content {
  flex-direction: row-reverse;
}

.message-bubble {
  padding: 16px;
  min-width: 0;
  max-width: 100%;
  background: #f8faff;
  border: 1px solid #e5e9ff;
  border-radius: 12px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.message.user .message-bubble {
  background: #eef2ff;
}

.message-content-wrapper {
  overflow: hidden;
  width: fit-content;
  min-width: 1.5em;
}

.markdown-body {
  font-size: 14px;
  line-height: 1.6;
  color: #1e293b;
}

/* 调整 Markdown 样式 */
.markdown-body :deep(strong) {
  font-weight: 600;
  color: #4338ca;
}

.markdown-body :deep(em) {
  color: #6366f1;
}

.markdown-body :deep(h1), 
.markdown-body :deep(h2), 
.markdown-body :deep(h3), 
.markdown-body :deep(h4), 
.markdown-body :deep(h5), 
.markdown-body :deep(h6) {
  margin: 1.5em 0 0.5em;
  font-weight: 600;
  line-height: 1.25;
  color: #1e293b;
}

.markdown-body :deep(h1:first-child),
.markdown-body :deep(h2:first-child),
.markdown-body :deep(h3:first-child),
.markdown-body :deep(h4:first-child),
.markdown-body :deep(h5:first-child),
.markdown-body :deep(h6:first-child) {
  margin-top: 0;
}

.markdown-body :deep(pre) {
  margin: 1em 0;
  padding: 16px;
  background: #f8fafc !important;
  border: 1px solid #e5e9ff;
  border-radius: 8px;
  position: relative;
  overflow: hidden;
}

.markdown-body :deep(code) {
  font-family: 'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 13px;
  line-height: 1.5;
}

.markdown-body :deep(:not(pre) > code) {
  padding: 2px 6px;
  border-radius: 4px;
  background: rgba(99, 102, 241, 0.1);
  color: #4338ca;
  font-size: 13px;
  border: none;
}

.markdown-body :deep(blockquote) {
  margin: 1em -16px;
  padding: 8px 16px;
  background: rgba(99, 102, 241, 0.05);
  border-left: 4px solid #818cf8;
  color: #4338ca;
}

.markdown-body :deep(ul), 
.markdown-body :deep(ol) {
  margin: 0.5em 0;
  padding-left: 1.5em;
  color: #334155;
}

.markdown-body :deep(p) {
  margin: 0;
  color: #334155;
}

.markdown-body :deep(p + p) {
  margin-top: 1em;
}

.markdown-body :deep(hr) {
  margin: 1.5em -16px;
  border: none;
  border-top: 1px solid #e5e9ff;
}

.markdown-body :deep(a) {
  color: #4f46e5;
  text-decoration: none;
  border-bottom: 1px solid rgba(99, 102, 241, 0.2);
}

.markdown-body :deep(a:hover) {
  border-bottom-color: #4f46e5;
}

/* 表格样式优化 */
.markdown-body :deep(table) {
  margin: 1em -16px;
  width: calc(100% + 32px);
  border-collapse: collapse;
}

.markdown-body :deep(th),
.markdown-body :deep(td) {
  padding: 8px 16px;
  border: 1px solid #e5e9ff;
  font-size: 13px;
}

.markdown-body :deep(th) {
  background: rgba(99, 102, 241, 0.05);
  font-weight: 500;
  color: #4338ca;
}

.markdown-body :deep(td) {
  background: white;
}

/* 列表项优化 */
.markdown-body :deep(li) {
  margin: 0.25em 0;
}

/* 代码块语言标识和复制按钮容器 */
.markdown-body :deep(pre::before) {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 32px;
  background: #f1f5f9;
  border-bottom: 1px solid #e5e9ff;
}

/* 语言标识 */
.markdown-body :deep(pre::after) {
  content: attr(class);
  position: absolute;
  top: 0;
  left: 12px;
  color: #64748b;
  font-size: 12px;
  font-family: system-ui, -apple-system, sans-serif;
  line-height: 32px;
  text-transform: uppercase;
}

/* 代码内容区域 */
.markdown-body :deep(pre code) {
  display: block;
  margin-top: 32px;
  padding: 0;
  overflow-x: auto;
}

/* 复制按钮 */
.markdown-body :deep(pre .code-copy-btn) {
  position: absolute;
  top: 4px;
  right: 4px;
  padding: 4px 8px;
  font-size: 12px;
  color: #64748b;
  background: white;
  border: 1px solid #e5e9ff;
  border-radius: 4px;
  cursor: pointer;
  opacity: 0;
  transition: all 0.2s ease;
}

.markdown-body :deep(pre:hover .code-copy-btn) {
  opacity: 1;
}

.markdown-body :deep(pre .code-copy-btn:hover) {
  color: #4f46e5;
  border-color: #818cf8;
  background: #f8faff;
}

/* 代码块滚动条 */
.markdown-body :deep(pre code::-webkit-scrollbar) {
  height: 4px;
}

.markdown-body :deep(pre code::-webkit-scrollbar-thumb) {
  background: #cbd5e1;
  border-radius: 2px;
}

.markdown-body :deep(pre code::-webkit-scrollbar-track) {
  background: transparent;
}

/* 修改 highlight.js 主题颜色 */
.markdown-body :deep(.hljs) {
  background: transparent;
  color: #334155;
}

.markdown-body :deep(.hljs-keyword),
.markdown-body :deep(.hljs-title.function_) {
  color: #4f46e5;
}

.markdown-body :deep(.hljs-string) {
  color: #0ea5e9;
}

.markdown-body :deep(.hljs-comment) {
  color: #94a3b8;
}

.markdown-body :deep(.hljs-variable),
.markdown-body :deep(.hljs-attr) {
  color: #0284c7;
}

.markdown-body :deep(.hljs-number) {
  color: #0891b2;
}

.markdown-body :deep(.hljs-title.class_) {
  color: #6366f1;
}

/* 添加消息时间 */
.message-time {
  font-size: 11px;
  color: #94a3b8;
  margin-top: 4px;
  padding: 0 12px;
}

/* 优化滚动条样式 */
.chat-messages::-webkit-scrollbar {
  width: 5px;
}

.chat-messages::-webkit-scrollbar-thumb {
  background: #e2e8f0;
  border-radius: 3px;
}

.chat-messages::-webkit-scrollbar-track {
  background: transparent;
}

.chat-messages:hover::-webkit-scrollbar-thumb {
  background: #cbd5e1;
}

.input-area {
  border-top: 1px solid #e5e9ff;
  padding: 16px 24px;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
}

.input-container {
  position: relative;
  display: grid;
  grid-template-columns: 1fr auto;
  grid-auto-rows: auto;
  column-gap: 12px;
  row-gap: 8px;
  align-items: stretch;
}

.input-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: space-between;
  gap: 8px;
  height: 100%;
  align-self: stretch;
}

/* 不再需要单独的第二行容器 */

:deep(.el-textarea__inner) {
  border: 1px solid #e5e9ff;
  background: #f8faff;
  min-height: 44px !important;
  max-height: 200px;
  padding: 12px 16px;
  font-size: 14px;
  line-height: 1.5;
  border-radius: 10px;
  transition: all 0.2s ease;
  resize: none;
  overflow-y: auto;
}

:deep(.el-textarea__inner:hover) {
  border-color: #818cf8;
  background: white;
}

:deep(.el-textarea__inner:focus) {
  border-color: #4f46e5;
  background: white;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.send-button {
  background: #eef2ff;
  color: #4f46e5;
  border: 1px solid #e5e9ff;
  height: 44px;
  min-width: 88px;
  border-radius: 10px;
  padding: 0 20px;
  font-size: 14px;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.message-input {
  width: 100%;
}

.send-button:not(:disabled):hover {
  background: #4f46e5;
  color: white;
  border: none;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(79, 70, 229, 0.15);
}

.send-button:disabled {
  background: #f8faff;
  border: 1px solid #e5e9ff;
  color: #94a3b8;
  cursor: not-allowed;
  opacity: 0.7;
}

.send-button .el-icon {
  font-size: 16px;
  margin-top: -1px;
}

.input-tips {
  margin-top: 8px;
  padding: 0 4px;
  color: #94a3b8;
  font-size: 12px;
}

.message-avatar .el-avatar {
  background: #f8faff;
  color: #64748b;
  font-weight: 500;
  border: 1px solid #e5e9ff;
}

.message-avatar .el-avatar.assistant {
  background: #f8faff;
  color: #4f46e5;
  border: 1px solid #e5e9ff;
}

.message-avatar .el-avatar.user {
  background: #eef2ff;
  color: #4f46e5;
  border: 1px solid #e5e9ff;
}

/* 消息操作行（底部一行，不再悬浮） */
.message-actions-row {
  display: flex;
  gap: 6px;
  margin-top: 8px;
}
.icon-action-button {
  height: 28px;
  width: 28px;
  padding: 4px;
}

/* 取消下方工具行（按钮已与发送同行） */
</style> 