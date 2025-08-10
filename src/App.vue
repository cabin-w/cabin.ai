<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import AIChat from './components/AIChat.vue'
import { useDark, useToggle } from '@vueuse/core'

const settingsVisible = ref(false)
// 固定外链：请按需替换为你的真实地址
const GITHUB_URL = 'https://github.com/cabin-w/cabin.ai'
const GITHUB_PAGE = 'https://github.com/cabin-w/cabin.ai'
const BLOG_URL = 'https://cabin.ink/'
const SILICONFLOW_URL = 'https://cloud.siliconflow.cn/i/oce44hC4'
const DEFAULT_AVATAR_URL = 'https://cabin.ink/images/avatar.png'
const apiKey = ref('')
const promptText = ref('')
const modelName = ref('')
const userAvatar = ref('')
const chatHistory = ref([])
const currentChatId = ref(null)
const isDark = useDark()
const toggleDark = useToggle(isDark)

// 响应式：移动端侧边栏开合
const isMobile = ref(false)
const sidebarOpen = ref(true)

const updateIsMobile = () => {
  isMobile.value = window.innerWidth <= 768
  if (isMobile.value) {
    sidebarOpen.value = false
  } else {
    sidebarOpen.value = true
  }
}

// 从localStorage获取已保存的设置
const initSettings = () => {
  apiKey.value = localStorage.getItem('apiKey') || ''
  promptText.value = localStorage.getItem('promptText') || ''
  modelName.value = localStorage.getItem('modelName') || 'deepseek-ai/DeepSeek-R1-Distill-Qwen-32B'
  userAvatar.value = localStorage.getItem('userAvatar') || DEFAULT_AVATAR_URL
}

// 保存设置
const saveSettings = () => {
  localStorage.setItem('apiKey', apiKey.value)
  localStorage.setItem('promptText', promptText.value)
  localStorage.setItem('modelName', modelName.value)
  localStorage.setItem('userAvatar', userAvatar.value)
  settingsVisible.value = false
}

// 创建新对话
const createNewChat = () => {
  const newChat = {
    id: Date.now(),
    title: '新对话',
    messages: [],
    createdAt: new Date().toLocaleString()
  }
  chatHistory.value.unshift(newChat)
  currentChatId.value = newChat.id
  saveChatHistory()
}

// 选择对话
const selectChat = (chatId) => {
  currentChatId.value = chatId
}

// 更新对话内容
const updateChat = (messages) => {
  const chat = chatHistory.value.find(c => c.id === currentChatId.value)
  if (chat) {
    chat.messages = messages
    // 更新对话标题为第一条消息的前20个字符
    if (messages.length > 0) {
      chat.title = messages[0].content.slice(0, 20) + (messages[0].content.length > 20 ? '...' : '')
    }
    saveChatHistory()
  }
}

// 删除对话
const deleteChat = (chatId) => {
  const index = chatHistory.value.findIndex(c => c.id === chatId)
  if (index > -1) {
    chatHistory.value.splice(index, 1)
    if (currentChatId.value === chatId) {
      currentChatId.value = chatHistory.value[0]?.id || null
    }
    saveChatHistory()
  }
}

// 保存聊天历史到localStorage
const saveChatHistory = () => {
  localStorage.setItem('chatHistory', JSON.stringify(chatHistory.value))
}

// 初始化
onMounted(() => {
  initSettings()
  updateIsMobile()
  window.addEventListener('resize', updateIsMobile)
  // 加载聊天历史
  const savedHistory = localStorage.getItem('chatHistory')
  if (savedHistory) {
    chatHistory.value = JSON.parse(savedHistory)
    currentChatId.value = chatHistory.value[0]?.id || null
  } else {
    createNewChat()
  }
})

onUnmounted(() => {
  window.removeEventListener('resize', updateIsMobile)
})
</script>

<template>
  <div class="app-layout">
    <div :class="['sidebar', { 'mobile-open': isMobile && sidebarOpen }]">
      <div class="sidebar-header">
        <h2>Cabin.AI</h2>
        <el-button
          class="theme-toggle"
          text
          @click="toggleDark()"
        >
          <el-icon v-if="isDark"><Moon /></el-icon>
          <el-icon v-else><Sunny /></el-icon>
        </el-button>
      </div>
      <div class="sidebar-content">
        <el-button class="new-chat" @click="createNewChat">
          <el-icon><Plus /></el-icon>
          新对话
        </el-button>
        <div class="history-container">
          <!-- <div class="section-header">
            <el-icon><ChatDotRound /></el-icon>
            <span>历史记录</span>
          </div> -->
          <el-scrollbar>
            <div class="history-list">
              <div v-for="chat in chatHistory" 
                   :key="chat.id" 
                   :class="['history-item', { active: currentChatId === chat.id }]"
                   @click="selectChat(chat.id)"
              >
                <div class="history-item-content">
                  <span class="history-title">{{ chat.title }}</span>
                  <span class="history-date">{{ chat.createdAt }}</span>
                </div>
                <el-button 
                  class="delete-btn" 
                  type="text" 
                  @click.stop="deleteChat(chat.id)"
                >
                  <el-icon><Delete /></el-icon>
                </el-button>
              </div>
            </div>
          </el-scrollbar>
        </div>

        <!-- AI 模型配置部分 -->
        <div class="config-container">
          <div class="section-header">
            <el-icon><Setting /></el-icon>
            <span>AI 模型配置</span>
            <el-button 
              type="primary" 
              link 
              size="small"
              class="config-button"
              @click="settingsVisible = true"
            >
              配置设置
            </el-button>
          </div>
          <div class="config-tags">
            <div class="setting-text">
              {{ apiKey ? 'API Key 已配置' : '未配置 API Key' }}
            </div>
            <div class="setting-text model-text">
              <el-tooltip 
                v-if="modelName" 
                :content="modelName"
                placement="top"
              >
                <span class="model-name">
                  当前模型: {{ modelName.split('/').pop() }}
                </span>
              </el-tooltip>
              <span v-else>使用默认模型</span>
            </div>
            <div class="setting-text">
              {{ promptText ? '已配置自定义角色' : '使用默认角色' }}
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 主内容区 -->
    <div class="main-content">
      <el-button 
        v-if="isMobile"
        class="mobile-toggle"
        :class="{ open: sidebarOpen }"
        @click="sidebarOpen = !sidebarOpen"
        :title="sidebarOpen ? '收起菜单' : '展开菜单'"
      >
        <el-icon v-if="sidebarOpen"><Fold /></el-icon>
        <el-icon v-else><Expand /></el-icon>
      </el-button>
      <AIChat 
        :api-key="apiKey" 
        :prompt-text="promptText"
        :model-name="modelName"
        :user-avatar="userAvatar"
        :chat-id="currentChatId"
        :initial-messages="chatHistory.find(c => c.id === currentChatId)?.messages || []"
        @update-messages="updateChat"
      />
    </div>

    <div 
      v-if="isMobile && sidebarOpen" 
      class="mobile-overlay" 
      @click="sidebarOpen = false"
    ></div>

    <!-- 详细设置对话框 -->
    <el-dialog
      v-model="settingsVisible"
      title="AI 模型配置"
      :width="isMobile ? '92vw' : '600px'"
    >
      <el-form label-position="top">
        <!-- API Key 设置 -->
        <el-form-item label="API Key">
          <div class="prompt-description">
            设置您的 API Key，用于访问 AI 服务
          </div>
          <el-input
            v-model="apiKey"
            type="password"
            placeholder="请输入 API Key"
            show-password
          >
            <template #append>
              <el-tooltip content="API Key 将保存在本地" placement="top">
                <el-icon><Key /></el-icon>
              </el-tooltip>
            </template>
          </el-input>
        </el-form-item>

        <!-- 模型选择 -->
        <el-form-item label="模型选择">
          <div class="prompt-description">
            选择或输入要使用的模型名称
          </div>
          <el-input 
            v-model="modelName" 
            placeholder="请输入模型名称..."
          >
            <template #append>
              <el-tooltip content="默认: deepseek-ai/DeepSeek-R1-Distill-Qwen-32B" placement="top">
                <el-icon><QuestionFilled /></el-icon>
              </el-tooltip>
            </template>
          </el-input>
        </el-form-item>

        <!-- 角色设定 -->
        <el-form-item label="角色设定">
          <div class="prompt-description">
            设定 AI 助手的角色特征、对话风格和行为模式
          </div>
          <el-input 
            v-model="promptText" 
            type="textarea" 
            :rows="isMobile ? 5 : 8"
            placeholder="请输入角色设定提示词..."
          />
          <div class="setting-tips">
            留空则使用默认设置：基础助手角色
          </div>
        </el-form-item>

        <!-- 我的头像 -->
        <el-form-item label="设置头像">
          <div class="prompt-description">可填写单/双字作为头像文本，或使用图片URL（支持 http(s) 与 data:image）。</div>
          <el-input v-model="userAvatar" placeholder="例如：我 / 小秋 / https://example.com/me.png" >
            <template #append>
              <el-tooltip content="默认: https://cabin.ink/images/avatar.png" placement="top">
                <el-icon><QuestionFilled /></el-icon>
              </el-tooltip>
            </template>
          </el-input>
        </el-form-item>
      </el-form>

        <!-- 固定外链：GitHub 与 博客 -->
        <div class="section-header links-header">
          <span>Add Star / My Blog / Register API Key / Select Model</span>
        </div>
        <div class="links-grid">
          <el-link :href="GITHUB_URL" target="_blank" type="primary">
            <el-icon style="margin-right:6px"><Link /></el-icon>
            ⭐ star the repo
          </el-link>
          <el-link :href= "GITHUB_PAGE" target="_blank" type="primary">
            <el-icon style="margin-right:6px"><Link /></el-icon>
            github
          </el-link>
          <el-link :href="BLOG_URL" target="_blank" type="primary">
            <el-icon style="margin-right:6px"><Link /></el-icon>
            cabin's blog
          </el-link>
          <el-link :href= "SILICONFLOW_URL" target="_blank" type="primary">
            <el-icon style="margin-right:6px"><Link /></el-icon>
            siliconflow
          </el-link>
        </div>

      <template #footer>
        <div class="dialog-footer">
          <el-button @click="settingsVisible = false">取消</el-button>
          <el-button type="primary" @click="saveSettings">
            保存设置
          </el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<style>
.app-layout {
  display: flex;
  height: 100vh;
  width: 100vw;
  background-color: #f8fafc;
  position: fixed;
  top: 0;
  left: 0;
}

.sidebar {
  width: 280px;
  background-color: #fff;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.05);
}

/* 移动端：侧边栏抽屉化 */
@media (max-width: 768px) {
  .sidebar {
    position: fixed;
    z-index: 20;
    top: 0;
    bottom: 0;
    left: -300px;
    transition: left 0.2s ease;
    width: 260px;
  }
  .sidebar.mobile-open {
    left: 0;
  }
}

.sidebar-header {
  padding: 20px 24px;
  background: var(--el-bg-color);
  border-bottom: 1px solid var(--el-border-color);
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 64px;
}

.sidebar-header h2 {
  margin: 0;
  font-size: 20px;
  background: linear-gradient(135deg, #4f46e5, #818cf8);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-weight: 700;
  letter-spacing: 0.5px;
}

.sidebar-content {
  padding: 12px;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow: hidden;
}

.new-chat {
  width: 100%;
  margin-bottom: 4px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  color: #4f46e5;
  transition: all 0.2s ease;
  height: 40px;
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  border-radius: 8px;
}

.new-chat:hover {
  background: #f1f5f9;
  border-color: #818cf8;
  transform: translateY(-1px);
}

.new-chat .el-icon {
  font-size: 16px;
  margin-top: -2px;
}

.history-container {
  flex: 1;
  overflow: hidden;
  margin: 0 -12px;
}

.history-list {
  padding: 4px 8px;
}

.history-item {
  padding: 10px 16px;
  margin: 2px 0;
  cursor: pointer;
  color: #64748b;
  transition: all 0.2s ease;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-radius: 6px;
}

.history-item:hover {
  background-color: #f1f5f9;
}

.history-item.active {
  background-color: #eff6ff;
  color: #4f46e5;
  font-weight: 500;
}

.history-item-content {
  flex: 1;
  overflow: hidden;
  margin-right: 8px;
}

.history-title {
  display: block;
  font-size: 13px;
  line-height: 1.4;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.history-date {
  display: block;
  font-size: 11px;
  color: #94a3b8;
  margin-top: 2px;
}

.delete-btn {
  opacity: 0;
  padding: 4px;
  height: 24px;
  width: 24px;
  min-width: 24px;
  transition: all 0.2s ease;
  color: #94a3b8;
  border-radius: 4px;
}

.history-item:hover .delete-btn {
  opacity: 1;
}

.delete-btn:hover {
  color: #ef4444;
  background-color: #fee2e2;
}

.main-content {
  flex: 1;
  overflow: hidden;
  background: linear-gradient(135deg, #fafafa, #f5f5f5);
}

/* 移动端：主内容层叠按钮与遮罩 */
@media (max-width: 768px) {
  .main-content {
    position: relative;
  }
  .mobile-toggle {
    position: absolute;
    top: max(12px, env(safe-area-inset-top));
    right: 12px;
    z-index: 25;
    height: 36px;
    min-width: 36px;
    padding: 0 10px;
    border-radius: 18px;
    background: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(8px);
    border: 1px solid #e5e7eb;
    color: #64748b;
    box-shadow: 0 4px 10px rgba(0,0,0,0.08);
    transition: all .2s ease;
  }
  .mobile-toggle:hover {
    background: #ffffff;
    color: #4f46e5;
    border-color: #c7d2fe;
  }
  .mobile-toggle.open {
    color: #4f46e5;
    border-color: #c7d2fe;
    background: rgba(255,255,255,0.95);
  }
  .mobile-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.35);
    z-index: 15;
  }
}

/* 夜间主题下的移动按钮样式 */
@media (max-width: 768px) {
  html.dark .mobile-toggle {
    background: rgba(26, 27, 30, 0.85);
    border-color: var(--chat-border-color);
    color: var(--chat-text-color);
    box-shadow: 0 4px 12px rgba(0,0,0,0.35);
  }
  html.dark .mobile-toggle:hover {
    background: rgba(26, 27, 30, 0.95);
    color: #818cf8;
    border-color: #818cf8;
  }
  html.dark .mobile-toggle.open {
    color: #818cf8;
    border-color: #818cf8;
    background: rgba(26, 27, 30, 0.95);
  }
}

.config-container {
  margin: 8px -12px 0;
  border-top: 1px solid #e2e8f0;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  color: #64748b;
  font-size: 13px;
  font-weight: 500;
}

.links-header {
  padding-left: 0;
}

.section-header .el-icon {
  font-size: 16px;
}

.section-header .el-button {
  margin-left: auto;
  padding: 4px 8px;
  font-size: 12px;
}

.config-tags {
  padding: 8px 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.setting-text {
  font-size: 12px;
  color: #64748b;
  padding: 2px 0;
}

.model-text {
  max-width: 200px;
}

.model-name {
  display: inline-block;
  max-width: 180px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  vertical-align: bottom;
}

/* 修改暗色主题相关样式 */
html.dark {
  --chat-bg-color: #1a1b1e;
  --chat-border-color: #2c2c30;
  --chat-bubble-bg: #2c2c30;
  --chat-bubble-user-bg: #313136;
  --chat-text-color: #cbd5e1;
  --chat-secondary-text: #94a3b8;
  --chat-code-bg: #1f1f23;
  --chat-code-border: #2c2c30;
}

html.dark .app-layout {
  background-color: var(--chat-bg-color);
}

html.dark .main-content {
  background: #1a1b1e;
}

html.dark .sidebar {
  background-color: var(--el-bg-color);
  border-right-color: var(--chat-border-color);
  box-shadow: none;
}

html.dark .sidebar-header h2 {
  background: linear-gradient(135deg, #818cf8, #6366f1);
  -webkit-background-clip: text;
  color: #818cf8;
  -webkit-text-fill-color: #818cf8;
}

html.dark .new-chat {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: #818cf8;
}

html.dark .new-chat:hover {
  background: var(--chat-bubble-user-bg);
  border-color: #818cf8;
}

html.dark .history-item {
  color: var(--chat-text-color);
}

html.dark .history-item:hover {
  background-color: var(--chat-bubble-bg);
}

html.dark .history-item.active {
  background-color: var(--chat-bubble-user-bg);
  color: #818cf8;
}

html.dark .history-date {
  color: var(--chat-secondary-text);
}

html.dark .delete-btn {
  color: var(--chat-text-color);
}

html.dark .delete-btn:hover {
  background-color: rgba(239, 68, 68, 0.2);
  color: #ef4444;
}

html.dark .sidebar-settings {
  background: var(--el-bg-color);
  border-color: var(--chat-border-color);
}

html.dark .settings-section {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark .settings-header {
  color: var(--chat-text-color);
}

html.dark :deep(.el-tag) {
  background: var(--chat-bubble-user-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark :deep(.el-dialog) {
  background: var(--el-bg-color);
}

html.dark :deep(.el-dialog__header) {
  border-color: var(--chat-border-color);
}

html.dark :deep(.el-dialog__footer) {
  border-color: var(--chat-border-color);
}

html.dark .prompt-description {
  color: var(--chat-text-color);
}

html.dark :deep(.el-scrollbar__thumb) {
  background-color: #4b5563;
}

html.dark :deep(.el-input-group__append) {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark .message-bubble {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
}

html.dark .message.user .message-bubble {
  background: var(--chat-bubble-user-bg);
}

html.dark .markdown-body {
  color: var(--chat-text-color);
}

html.dark .markdown-body :deep(h1),
html.dark .markdown-body :deep(h2),
html.dark .markdown-body :deep(h3),
html.dark .markdown-body :deep(h4),
html.dark .markdown-body :deep(h5),
html.dark .markdown-body :deep(h6) {
  color: var(--chat-text-color);
}

html.dark .markdown-body :deep(strong) {
  color: #818cf8;
}

html.dark .markdown-body :deep(em) {
  color: #93c5fd;
}

html.dark .markdown-body :deep(blockquote) {
  background: var(--chat-bubble-user-bg);
  border-left-color: #818cf8;
  color: #93c5fd;
}

html.dark .markdown-body :deep(a) {
  color: #818cf8;
  border-bottom-color: rgba(129, 140, 248, 0.2);
}

html.dark .markdown-body :deep(a:hover) {
  border-bottom-color: #818cf8;
}

html.dark .markdown-body :deep(pre) {
  background: var(--chat-code-bg) !important;
  border-color: var(--chat-code-border);
}

html.dark .markdown-body :deep(pre::before) {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
}

html.dark .markdown-body :deep(pre .code-copy-btn) {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark .markdown-body :deep(pre .code-copy-btn:hover) {
  background: var(--chat-bubble-user-bg);
  border-color: #818cf8;
  color: #818cf8;
}

html.dark .markdown-body :deep(code) {
  color: #e5e7eb;
}

html.dark .markdown-body :deep(:not(pre) > code) {
  background: var(--chat-bubble-user-bg);
  color: #818cf8;
}

html.dark .input-area {
  background: var(--el-bg-color);
  border-color: var(--chat-border-color);
}

html.dark :deep(.el-textarea__inner) {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark :deep(.el-textarea__inner:hover) {
  border-color: #818cf8;
  background: var(--chat-bubble-user-bg);
}

html.dark .send-button {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: #818cf8;
}

html.dark .send-button:hover:not(:disabled) {
  background: #818cf8;
  color: white;
}

html.dark .message-avatar .el-avatar {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark .message-avatar .el-avatar.assistant {
  color: #818cf8;
}

html.dark .message-avatar .el-avatar.user {
  background: var(--chat-bubble-user-bg);
  color: #818cf8;
}

/* 对话框和设置面板的暗色主题 */
html.dark :deep(.el-dialog__title),
html.dark .settings-header,
html.dark .prompt-description {
  color: var(--chat-text-color);
}

html.dark :deep(.el-input__inner),
html.dark :deep(.el-textarea__inner) {
  color: var(--chat-text-color) !important;
}

html.dark :deep(.el-dialog__body) {
  color: var(--chat-text-color);
}

html.dark .history-item {
  color: var(--chat-text-color);
}

html.dark .history-date {
  color: var(--chat-secondary-text);
}

html.dark :deep(.el-input-group__append) {
  color: var(--chat-text-color);
}

html.dark :deep(.el-tag) {
  color: var(--chat-text-color);
}

html.dark :deep(.el-button) {
  color: var(--chat-text-color);
}

html.dark :deep(.el-button:not(.is-disabled):hover) {
  color: #818cf8;
}

html.dark :deep(.el-input__inner::placeholder),
html.dark :deep(.el-textarea__inner::placeholder) {
  color: var(--chat-secondary-text);
}

html.dark :deep(.el-dialog__footer) .el-button {
  border-color: var(--chat-border-color);
  background: var(--chat-bubble-bg);
}

html.dark :deep(.el-dialog__footer) .el-button:hover {
  border-color: #818cf8;
  color: #818cf8;
}

html.dark :deep(.el-dialog__footer) .el-button--primary {
  background: #818cf8;
  border-color: #818cf8;
  color: white;
}

html.dark :deep(.el-dialog__footer) .el-button--primary:hover {
  background: #6366f1;
  border-color: #6366f1;
  color: white;
}

/* 修改角色设定按钮样式 */
html.dark .settings-header .el-button {
  color: #818cf8;
}

html.dark .settings-header .el-button:hover {
  color: #6366f1;
}

/* 修改对话框中的链接按钮样式 */
html.dark :deep(.el-button.is-link) {
  color: var(--chat-text-color);
}

html.dark :deep(.el-button.is-link:hover) {
  color: #818cf8;
}

/* 修改对话框中的主要按钮样式 */
html.dark :deep(.el-button--primary) {
  background: #818cf8;
  border-color: #818cf8;
  color: white;
}

html.dark :deep(.el-button--primary:hover) {
  background: #6366f1;
  border-color: #6366f1;
}

/* 修改对话框中的普通按钮样式 */
html.dark :deep(.el-button) {
  background: var(--chat-bubble-bg);
  border-color: var(--chat-border-color);
  color: var(--chat-text-color);
}

html.dark :deep(.el-button:hover) {
  border-color: #818cf8;
  color: #818cf8;
}

/* 修改输入框的占位符颜色 */
html.dark :deep(.el-input__inner::placeholder),
html.dark :deep(.el-textarea__inner::placeholder) {
  color: var(--chat-secondary-text);
}

/* 添加新的样式 */
.setting-tips {
  font-size: 12px;
  color: var(--chat-secondary-text);
}

.prompt-description {
  color: var(--chat-text-color);
  font-size: 13px;
  margin-bottom: 8px;
  line-height: 1.5;
}

:deep(.el-form-item) {
  margin-bottom: 24px;
}

:deep(.el-form-item:last-child) {
  margin-bottom: 0;
}

:deep(.el-input-group__append) {
  padding: 0 12px;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

.links-grid {
  display: flex;
  align-items: center;
  gap: 12px 24px;
  flex-wrap: nowrap;
}
.links-grid :deep(.el-link) {
  white-space: nowrap;
}

/* 设置对话框在移动端的适配 */
@media (max-width: 768px) {
  :deep(.el-dialog) {
    margin: 10vh auto !important;
  }
  :deep(.el-dialog__body) {
    padding: 10px 16px !important;
  }
  :deep(.el-form-item) {
    margin-bottom: 16px !important;
  }
}

/* 暗色主题适配 */
html.dark .setting-tips {
  color: var(--chat-secondary-text);
}

html.dark .prompt-description {
  color: var(--chat-text-color);
}

.settings-info {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 8px;
}

.setting-tag {
  font-size: 12px;
}

.model-tag {
  max-width: 200px;
}

.model-name {
  display: inline-block;
  max-width: 180px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  vertical-align: bottom;
}

/* 暗色主题适配 */
html.dark .setting-tag {
  border-color: var(--chat-border-color) !important;
  color: var(--chat-text-color) !important;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  color: #64748b;
  font-size: 13px;
  font-weight: 500;
}

.section-header .el-icon {
  font-size: 16px;
}

.section-header .el-button {
  margin-left: auto;
  padding: 4px 8px;
  font-size: 12px;
}

.config-container {
  margin-top: 8px;
  border-top: 1px solid #e2e8f0;
}

.config-tags {
  padding: 8px 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.setting-tag {
  width: 100%;
  justify-content: flex-start;
}

/* 暗色主题适配 */
html.dark .section-header {
  color: var(--chat-text-color);
}

html.dark .config-container {
  border-color: var(--chat-border-color);
}

.config-button {
  color: #4f46e5 !important;
}

.config-button:hover {
  color: #6366f1 !important;
}

/* 暗色主题适配 */
html.dark .config-button {
  color: #818cf8 !important;
}

html.dark .config-button:hover {
  color: #6366f1 !important;
}
</style>
