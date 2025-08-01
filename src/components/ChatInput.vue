<template>
  <div class="chat-input-container">
    <!-- @ 提及选择器 -->
    <div
      v-if="showMentionSelector"
      class="mention-selector"
      :style="mentionSelectorStyle"
    >
      <div class="mention-header">
        <n-icon size="16">
          <svg viewBox="0 0 24 24">
            <path fill="currentColor" d="M12,2A10,10 0 0,0 2,12A10,10 0 0,0 12,22A10,10 0 0,0 22,12A10,10 0 0,0 12,2M7,9A2,2 0 0,1 9,7A2,2 0 0,1 11,9A2,2 0 0,1 9,11A2,2 0 0,1 7,9M15,9A2,2 0 0,1 17,7A2,2 0 0,1 19,9A2,2 0 0,1 17,11A2,2 0 0,1 15,9M12,17.5C14.33,17.5 16.31,16.04 17.11,14H6.89C7.69,16.04 9.67,17.5 12,17.5Z"/>
          </svg>
        </n-icon>
        <span>选择要提及的实例</span>
      </div>
      <div class="mention-list">
        <div
          v-for="(agent, index) in filteredAgents"
          :key="agent.name"
          :class="[
            'mention-item',
            { 'mention-item-selected': index === selectedMentionIndex }
          ]"
          @click="selectMention(agent)"
          @mouseenter="selectedMentionIndex = index"
        >
          <div class="mention-avatar">
            <n-icon size="20">
              <svg viewBox="0 0 24 24">
                <path fill="currentColor" d="M12,4A4,4 0 0,1 16,8A4,4 0 0,1 12,12A4,4 0 0,1 8,8A4,4 0 0,1 12,4M12,14C16.42,14 20,15.79 20,18V20H4V18C4,15.79 7.58,14 12,14Z"/>
              </svg>
            </n-icon>
          </div>
          <div class="mention-info">
            <div class="mention-name">{{ agent.name }}</div>
            <div class="mention-role">{{ agent.role }}</div>
          </div>
          <div class="mention-status">
            <div :class="['status-dot', `status-${agent.status}`]"></div>
            <span class="status-text">{{ getStatusText(agent.status) }}</span>
          </div>
        </div>
      </div>
      <div v-if="filteredAgents.length === 0" class="mention-empty">
        <n-icon size="24" color="var(--text-tertiary)">
          <svg viewBox="0 0 24 24">
            <path fill="currentColor" d="M12,2A10,10 0 0,0 2,12A10,10 0 0,0 12,22A10,10 0 0,0 22,12A10,10 0 0,0 12,2M12,4A8,8 0 0,1 20,12A8,8 0 0,1 12,20A8,8 0 0,1 4,12A8,8 0 0,1 12,4M11,9H13V7H11M11,17H13V11H11V17Z"/>
          </svg>
        </n-icon>
        <p>没有找到匹配的实例</p>
      </div>
    </div>

    <div
      class="chat-input"
      @dragover="handleDragOver"
      @dragleave="handleDragLeave"
      @drop="handleDrop"
      :class="{ 'drag-over': isDragOver }"
    >
      <!-- 拖拽提示 -->
      <div v-if="isDragOver" class="drag-overlay">
        <n-icon size="48" :color="'var(--color-success)'">
          <svg viewBox="0 0 24 24">
            <path
              fill="currentColor"
              d="M14,2H6A2,2 0 0,0 4,4V20A2,2 0 0,0 6,22H18A2,2 0 0,0 20,20V8L14,2M18,20H6V4H13V9H18V20Z"
            />
          </svg>
        </n-icon>
        <p>释放文件以上传</p>
      </div>

      <!-- 输入区域 -->
      <div class="input-area">
        <!-- 工具栏 -->
        <div class="toolbar">
          <!-- 文件上传按钮 -->
          <n-button
            text
            @click="handleFileUpload"
            class="attachment-button"
            :disabled="!isConnected"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M16.5,6V17.5A4,4 0 0,1 12.5,21.5A4,4 0 0,1 8.5,17.5V5A2.5,2.5 0 0,1 11,2.5A2.5,2.5 0 0,1 13.5,5V15.5A1,1 0 0,1 12.5,16.5A1,1 0 0,1 11.5,15.5V6H10V15.5A2.5,2.5 0 0,0 12.5,18A2.5,2.5 0 0,0 15,15.5V5A4,4 0 0,0 11,1A4,4 0 0,0 7,5V17.5A5.5,5.5 0 0,0 12.5,23A5.5,5.5 0 0,0 18,17.5V6H16.5Z"
                  />
                </svg>
              </n-icon>
            </template>
          </n-button>
        </div>

        <!-- 输入框 -->
        <n-input
          ref="inputRef"
          v-model:value="inputMessage"
          type="textarea"
          :placeholder="placeholderText"
          :autosize="{ minRows: 1, maxRows: 6 }"
          :disabled="!isConnected"
          @keydown="handleKeyDown"
          @paste="handlePaste"
          @input="handleInput"
          class="message-input"
        />

        <!-- 发送按钮 -->
        <div class="send-area">
          <!-- 消息长度指示器 -->
          <div 
            v-if="messageLength > 0" 
            class="message-length-indicator"
            :class="{
              'warning': messageSizeInfo.warningLevel === 'warning',
              'error': messageSizeInfo.warningLevel === 'danger'
            }"
          >
            <span class="length-text">
              {{ messageSizeInfo.charCount }}/{{ MESSAGE_LIMITS.MAX_CHARS }}
            </span>
            <span v-if="messageSizeInfo.byteSize > 1024" class="size-text">
              ({{ Math.round(messageSizeInfo.byteSize / 1024) }}KB)
            </span>
            <span v-if="!messageSizeInfo.isValid" class="error-text">
              过大
            </span>
          </div>
          
          <n-button
            type="primary"
            @click="handleSendMessage"
            :disabled="!canSendMessage"
            class="send-button"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path fill="currentColor" d="M2,21L23,12L2,3V10L17,12L2,14V21Z" />
                </svg>
              </n-icon>
            </template>
            发送
          </n-button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, nextTick, watch, onMounted, onUnmounted } from 'vue'
import { useMessage } from 'naive-ui'
import { formatFileSize } from '@/utils/file'
import { filesApi } from '@/api/files'
import { useUserStore } from '@/stores/user'
import { agentApi, type Agent } from '@/api/agents'
import { checkMessageSize, splitLongMessage, getMessageSizeWarningLevel, MESSAGE_LIMITS } from '@/utils/messageUtils'

const props = defineProps<{
  isConnected: boolean
  namespace?: string
}>()

const emit = defineEmits<{
  send: [message: string]
  'send-image': [url: string]
}>()

// 状态管理
const message = useMessage()
const userStore = useUserStore()

// 响应式数据
const inputMessage = ref('')
const inputRef = ref()
const isDragOver = ref(false)

// @ 功能相关状态
const showMentionSelector = ref(false)
const mentionQuery = ref('')
const selectedMentionIndex = ref(0)
const mentionStartPos = ref(0)
const agents = ref<Agent[]>([])
const mentionSelectorStyle = ref({})

// 计算属性
const canSendMessage = computed(() => {
  return inputMessage.value.trim() && props.isConnected
})

const placeholderText = computed(() => {
  if (!props.isConnected) {
    return '连接断开，无法发送消息...'
  }
  return '输入消息... (支持拖拽文件上传，输入@提及实例)'
})

// 消息长度统计
const messageLength = computed(() => {
  return inputMessage.value.length
})

const messageSizeInfo = computed(() => {
  const text = inputMessage.value
  const sizeCheck = checkMessageSize(text)
  const warningLevel = getMessageSizeWarningLevel(text)
  
  return {
    ...sizeCheck,
    warningLevel,
    charPercentage: (sizeCheck.charCount / MESSAGE_LIMITS.MAX_CHARS) * 100,
    bytePercentage: (sizeCheck.byteSize / MESSAGE_LIMITS.MAX_BYTES) * 100
  }
})

// 过滤的Agent列表
const filteredAgents = computed(() => {
  if (!mentionQuery.value) {
    return agents.value
  }
  
  const query = mentionQuery.value.toLowerCase()
  return agents.value.filter(agent => 
    agent.name.toLowerCase().includes(query) ||
    agent.role.toLowerCase().includes(query)
  )
})

// 获取状态文本
const getStatusText = (status: string) => {
  const statusMap = {
    running: '运行中',
    idle: '空闲',
    error: '错误',
    Creating: '创建中',
    Terminating: '终止中'
  }
  return statusMap[status] || status
}

// 加载Agent列表
const loadAgents = async () => {
  if (!props.namespace) return
  
  try {
    const response = await agentApi.getList(props.namespace)
    agents.value = response.items || []
  } catch (error) {
    console.error('加载实例列表失败:', error)
    agents.value = []
  }
}

// 处理输入事件
const handleInput = (value: string) => {
  checkMentionTrigger(value)
}

// 检查@触发
const checkMentionTrigger = (value: string) => {
  const input = getInputElement()
  if (!input) return

  const cursorPos = input.selectionStart || 0
  const textBeforeCursor = value.substring(0, cursorPos)
  
  // 查找最后一个@符号的位置
  const lastAtIndex = textBeforeCursor.lastIndexOf('@')
  
  if (lastAtIndex === -1) {
    hideMentionSelector()
    return
  }
  
  // 检查@符号前是否为空格或行首
  const charBeforeAt = lastAtIndex > 0 ? textBeforeCursor[lastAtIndex - 1] : ' '
  if (charBeforeAt !== ' ' && charBeforeAt !== '\n' && lastAtIndex !== 0) {
    hideMentionSelector()
    return
  }
  
  // 获取@后的查询文本
  const queryText = textBeforeCursor.substring(lastAtIndex + 1)
  
  // 检查查询文本是否包含空格（如果包含空格，说明已经完成了一个@提及）
  if (queryText.includes(' ') || queryText.includes('\n')) {
    hideMentionSelector()
    return
  }
  
  // 如果只有一个实例，不显示选择器
  if (agents.value.length <= 1) {
    hideMentionSelector()
    return
  }
  
  // 显示提及选择器
  mentionStartPos.value = lastAtIndex
  mentionQuery.value = queryText
  selectedMentionIndex.value = 0
  showMentionSelector.value = true
  
  // 计算选择器位置
  nextTick(() => {
    updateMentionSelectorPosition()
  })
}

// 获取输入框元素
const getInputElement = () => {
  if (!inputRef.value) return null
  
  return (
    inputRef.value.inputElRef ||
    inputRef.value.textareaElRef ||
    inputRef.value.$el?.querySelector('textarea') ||
    inputRef.value.$el?.querySelector('input')
  )
}

// 更新选择器位置
const updateMentionSelectorPosition = () => {
  const input = getInputElement()
  if (!input) return
  
  // 简单的位置计算，将选择器放在输入框上方
  const inputRect = input.getBoundingClientRect()
  mentionSelectorStyle.value = {
    position: 'fixed',
    bottom: `${window.innerHeight - inputRect.top + 8}px`,
    left: `${inputRect.left}px`,
    width: `${Math.min(320, inputRect.width)}px`,
    zIndex: 1000
  }
}

// 选择提及
const selectMention = (agent: Agent) => {
  const currentValue = inputMessage.value
  const beforeMention = currentValue.substring(0, mentionStartPos.value)
  const afterMention = currentValue.substring(mentionStartPos.value + 1 + mentionQuery.value.length)
  
  // 插入@提及
  const newValue = `${beforeMention}@${agent.name} ${afterMention}`
  inputMessage.value = newValue
  
  // 设置光标位置
  nextTick(() => {
    const input = getInputElement()
    if (input) {
      const newCursorPos = mentionStartPos.value + agent.name.length + 2 // @name + space
      input.setSelectionRange(newCursorPos, newCursorPos)
      input.focus()
    }
  })
  
  hideMentionSelector()
}

// 隐藏提及选择器
const hideMentionSelector = () => {
  showMentionSelector.value = false
  mentionQuery.value = ''
  selectedMentionIndex.value = 0
}

// 处理键盘事件
const handleKeyDown = (e: KeyboardEvent) => {
  // 如果提及选择器显示，处理导航键
  if (showMentionSelector.value) {
    switch (e.key) {
      case 'ArrowUp':
        e.preventDefault()
        selectedMentionIndex.value = Math.max(0, selectedMentionIndex.value - 1)
        break
      case 'ArrowDown':
        e.preventDefault()
        selectedMentionIndex.value = Math.min(
          filteredAgents.value.length - 1,
          selectedMentionIndex.value + 1
        )
        break
      case 'Enter':
        e.preventDefault()
        if (filteredAgents.value[selectedMentionIndex.value]) {
          selectMention(filteredAgents.value[selectedMentionIndex.value])
        }
        break
      case 'Escape':
        e.preventDefault()
        hideMentionSelector()
        break
      case 'Tab':
        e.preventDefault()
        if (filteredAgents.value[selectedMentionIndex.value]) {
          selectMention(filteredAgents.value[selectedMentionIndex.value])
        }
        break
      default:
        // 其他键继续正常处理
        break
    }
    return
  }
  
  // 正常的发送消息处理
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    handleSendMessage()
  }
}

// 发送消息
const handleSendMessage = async () => {
  if (!canSendMessage.value) return

  const text = inputMessage.value.trim()
  if (!text) return

  try {
    // 直接发送消息
    emit('send', text)
    inputMessage.value = ''
    hideMentionSelector()
  } catch (error) {
    console.error('❌ 发送过程中出错:', error)
  }
}

// 监听namespace变化，重新加载Agent列表
watch(
  () => props.namespace,
  (newNamespace) => {
    if (newNamespace) {
      loadAgents()
    }
  },
  { immediate: true }
)

// 点击外部隐藏选择器
const handleClickOutside = (e: Event) => {
  if (showMentionSelector.value) {
    const target = e.target as Element
    const selector = document.querySelector('.mention-selector')
    const input = getInputElement()
    
    if (selector && !selector.contains(target) && input && !input.contains(target)) {
      hideMentionSelector()
    }
  }
}

// 生命周期
onMounted(() => {
  document.addEventListener('click', handleClickOutside)
  if (props.namespace) {
    loadAgents()
  }
})

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside)
})

// 上传文件并插入链接到输入框
const uploadAndInsertFile = async (file: File) => {
  try {
    console.log('📤 开始上传文件:', file.name, file.type, formatFileSize(file.size))

    // 显示上传进度提示
    const loadingMessage = message.loading(`正在上传 ${file.name}...`, { duration: 0 })

    // 上传文件
    const result = await filesApi.uploadFile(userStore.currentUser.username, file)
    console.log('✅ 文件上传成功:', result)

    // 关闭加载提示
    loadingMessage.destroy()

    // 根据文件类型生成不同的链接格式
    const fileLink = generateFileLink(file, result.url)

    // 插入到输入框中
    insertTextAtCursor(fileLink)

    message.success(`文件 ${file.name} 上传成功`)
  } catch (error) {
    console.error('❌ 上传文件失败:', error)
    message.error(`上传文件 ${file.name} 失败: ${error.message}`)
  }
}

// 生成文件链接格式
const generateFileLink = (file: File, url: string) => {
  const fileName = file.name
  const fileType = file.type

  // 判断文件类型并生成相应格式
  if (fileType.startsWith('image/')) {
    return `[图片]${url}`
  } else if (fileType.startsWith('video/')) {
    return `[视频]${url}`
  } else if (fileType.startsWith('audio/')) {
    return `[音频]${url}`
  } else if (fileType.includes('pdf')) {
    return `[PDF]${url}`
  } else if (fileType.includes('word') || fileName.endsWith('.doc') || fileName.endsWith('.docx')) {
    return `[文档]${url}`
  } else if (
    fileType.includes('excel') ||
    fileName.endsWith('.xls') ||
    fileName.endsWith('.xlsx')
  ) {
    return `[表格]${url}`
  } else if (
    fileType.includes('powerpoint') ||
    fileName.endsWith('.ppt') ||
    fileName.endsWith('.pptx')
  ) {
    return `[演示]${url}`
  } else if (fileType.includes('zip') || fileType.includes('rar') || fileType.includes('7z')) {
    return `[压缩包]${url}`
  } else {
    return `[文件]${url}`
  }
}

// 在光标位置插入文本
const insertTextAtCursor = (text: string) => {
  console.log('🔧 尝试插入文本:', text)

  try {
    // 尝试多种方式访问input元素
    let input = null

    if (inputRef.value) {
      console.log('📝 inputRef存在，尝试获取DOM元素')
      // 尝试不同的访问路径
      input =
        inputRef.value.inputElRef ||
        inputRef.value.textareaElRef ||
        inputRef.value.$el?.querySelector('textarea') ||
        inputRef.value.$el?.querySelector('input')

      console.log('📝 获取到的input元素:', input)
    }

    if (input && typeof input.selectionStart === 'number') {
      console.log('✅ 找到有效的input元素，使用光标位置插入')
      const start = input.selectionStart
      const end = input.selectionEnd
      const currentValue = inputMessage.value

      // 在光标位置插入文本
      const newValue = currentValue.substring(0, start) + text + currentValue.substring(end)
      inputMessage.value = newValue

      // 设置新的光标位置
      nextTick(() => {
        try {
          const newCursorPos = start + text.length
          input.setSelectionRange(newCursorPos, newCursorPos)
          input.focus()
          console.log('✅ 文本已插入到光标位置，新位置:', newCursorPos)
        } catch (error) {
          console.warn('设置光标位置失败:', error)
        }
      })
      return
    }

    // 备用方案：追加到末尾
    console.log('⚠️ 无法获取光标位置，使用追加方案')
    appendTextToEnd(text)
  } catch (error) {
    console.error('插入文本失败:', error)
    // 最后的备用方案
    appendTextToEnd(text)
  }
}

// 追加文本到末尾的辅助函数
const appendTextToEnd = (text: string) => {
  console.log('📝 追加文本到末尾:', text)

  // 确保有适当的分隔符
  if (inputMessage.value) {
    const lastChar = inputMessage.value.slice(-1)
    if (lastChar !== ' ' && lastChar !== '\n') {
      inputMessage.value += ' '
    }
  }

  inputMessage.value += text

  // 尝试聚焦输入框
  nextTick(() => {
    try {
      if (inputRef.value) {
        if (typeof inputRef.value.focus === 'function') {
          inputRef.value.focus()
        }
      }
      console.log('✅ 文本已追加到末尾')
    } catch (error) {
      console.warn('聚焦输入框失败:', error)
    }
  })
}

// 处理文件上传按钮点击
const handleFileUpload = () => {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = '*'
  input.multiple = true
  input.onchange = async (e) => {
    const selectedFiles = Array.from((e.target as HTMLInputElement).files || [])
    console.log('📁 选择文件数量:', selectedFiles.length)

    for (const file of selectedFiles) {
      await uploadAndInsertFile(file)
    }
  }
  input.click()
}

// 处理图片上传按钮点击 (暂未使用)
// const handleImageUpload = () => {
//   const input = document.createElement('input')
//   input.type = 'file'
//   input.accept = 'image/*'
//   input.multiple = true
//   input.onchange = async (e) => {
//     const selectedImages = Array.from((e.target as HTMLInputElement).files || [])
//     console.log('🖼️ 选择图片数量:', selectedImages.length)

//     for (const file of selectedImages) {
//       await uploadAndInsertFile(file)
//     }
//   }
//   input.click()
// }

// 处理粘贴事件
const handlePaste = async (e: ClipboardEvent) => {
  if (!e.clipboardData) return

  const items = Array.from(e.clipboardData.items)
  const fileItems = items.filter((item) => item.kind === 'file')

  for (const fileItem of fileItems) {
    e.preventDefault()
    const file = fileItem.getAsFile()
    if (file) {
      await uploadAndInsertFile(file)
    }
  }
}

// 处理拖拽事件
const handleDragOver = (e: DragEvent) => {
  e.preventDefault()
  isDragOver.value = true
}

const handleDragLeave = (e: DragEvent) => {
  e.preventDefault()
  if (!(e.currentTarget as Element)?.contains(e.relatedTarget as Node)) {
    isDragOver.value = false
  }
}

const handleDrop = async (e: DragEvent) => {
  e.preventDefault()
  isDragOver.value = false

  const droppedFiles = Array.from(e.dataTransfer?.files || [])
  console.log('🗂️ 拖拽文件数量:', droppedFiles.length)

  for (const file of droppedFiles) {
    await uploadAndInsertFile(file)
  }
}
</script>

<style scoped lang="scss">
.chat-input-container {
  position: relative;
}

// @ 提及选择器样式
.mention-selector {
  background-color: var(--bg-primary);
  border: 1px solid var(--border-primary);
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(8px);
  max-height: 280px;
  overflow: hidden;
  animation: mentionFadeIn 0.2s ease-out;

  .mention-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 16px;
    background-color: var(--bg-secondary);
    border-bottom: 1px solid var(--border-primary);
    color: var(--text-secondary);
    font-size: 13px;
    font-weight: 500;
  }

  .mention-list {
    max-height: 200px;
    overflow-y: auto;
    padding: 4px 0;

    .mention-item {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 8px 16px;
      cursor: pointer;
      transition: all 0.2s ease;
      border-left: 3px solid transparent;

      &:hover,
      &.mention-item-selected {
        background-color: var(--bg-hover);
        border-left-color: var(--color-primary);
      }

      .mention-avatar {
        width: 32px;
        height: 32px;
        border-radius: 6px;
        background-color: var(--bg-tertiary);
        display: flex;
        align-items: center;
        justify-content: center;
        color: var(--text-secondary);
        flex-shrink: 0;
      }

      .mention-info {
        flex: 1;
        min-width: 0;

        .mention-name {
          font-weight: 500;
          color: var(--text-primary);
          font-size: 14px;
          margin-bottom: 2px;
        }

        .mention-role {
          font-size: 12px;
          color: var(--text-tertiary);
          overflow: hidden;
          text-overflow: ellipsis;
          white-space: nowrap;
        }
      }

      .mention-status {
        display: flex;
        align-items: center;
        gap: 6px;
        flex-shrink: 0;

        .status-dot {
          width: 8px;
          height: 8px;
          border-radius: 50%;
          
          &.status-running {
            background-color: var(--color-success);
            box-shadow: 0 0 4px rgba(var(--color-success-rgb), 0.4);
          }
          
          &.status-idle {
            background-color: var(--color-warning);
          }
          
          &.status-error {
            background-color: var(--color-error);
          }
          
          &.status-Creating,
          &.status-Terminating {
            background-color: var(--color-info);
            animation: pulse 1.5s infinite;
          }
        }

        .status-text {
          font-size: 11px;
          color: var(--text-tertiary);
          font-weight: 500;
        }
      }
    }
  }

  .mention-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 32px 16px;
    color: var(--text-tertiary);

    p {
      margin: 8px 0 0 0;
      font-size: 14px;
    }
  }
}

@keyframes mentionFadeIn {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.chat-input {
  background-color: var(--bg-primary);
  border-top: 1px solid var(--border-primary);
  color: var(--text-primary);
  position: relative;
  transition: all 0.3s ease;

  &.drag-over {
    background-color: rgba(16, 185, 129, 0.05);
    border-color: var(--color-success);
  }
}

.drag-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(7, 193, 96, 0.1);
  backdrop-filter: blur(4px);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 10;
  border: 2px dashed var(--color-success);
  border-radius: 8px;
  margin: 8px;

  p {
    margin: 12px 0 0 0;
    color: var(--color-success);
    font-weight: 500;
    font-size: 16px;
  }
}

.input-area {
  display: flex;
  align-items: flex-end;
  gap: 12px;
  padding: 12px 16px;
  background-color: var(--bg-primary);
}

.toolbar {
  display: flex;
  flex-direction: column;
  gap: 4px;

  .attachment-button,
  .image-button {
    width: 36px;
    height: 36px;
    border-radius: 6px;
    transition: all 0.2s ease;
    color: var(--text-secondary);

    &:hover {
      background-color: var(--bg-hover);
      color: var(--text-primary);
    }

    &:disabled {
      opacity: 0.5;
      color: var(--text-disabled);
    }
  }
}

.message-input {
  flex: 1;

  // 重写Naive UI输入框样式以适配主题
  :deep(.n-input) {
    background-color: var(--bg-secondary) !important;
    border: 1px solid var(--border-primary) !important;
    border-radius: 8px !important;
    transition: all 0.3s ease !important;

    // 输入框内部元素
    .n-input__input-el,
    .n-input__textarea-el {
      background-color: transparent !important;
      color: var(--text-primary) !important;
      border: none !important;
      font-size: 14px !important;
      line-height: 1.5 !important;
      padding: 8px 12px !important;

      &::placeholder {
        color: var(--text-tertiary) !important;
        opacity: 1 !important;
      }

      &:focus {
        outline: none !important;
      }
    }

    // 边框状态
    .n-input__border,
    .n-input__state-border {
      border: none !important;
    }

    // 悬停状态
    &:hover {
      border-color: var(--border-focus) !important;
      background-color: var(--bg-hover) !important;
    }

    // 聚焦状态
    &.n-input--focus {
      border-color: var(--color-primary) !important;
      background-color: var(--bg-secondary) !important;
      box-shadow: 0 0 0 2px rgba(var(--color-primary-rgb), 0.1) !important;
    }

    // 禁用状态
    &.n-input--disabled {
      background-color: var(--bg-tertiary) !important;
      border-color: var(--border-secondary) !important;
      opacity: 0.6 !important;

      .n-input__input-el,
      .n-input__textarea-el {
        color: var(--text-disabled) !important;
      }
    }
  }

  // textarea特殊处理
  :deep(.n-input__textarea-el) {
    resize: none !important;
    min-height: 36px !important;
    max-height: 144px !important;
  }
}

.send-area {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;

  .message-length-indicator {
    font-size: 11px;
    color: var(--text-tertiary);
    padding: 2px 6px;
    border-radius: 4px;
    background-color: var(--bg-tertiary);
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    gap: 4px;

    &.warning {
      color: var(--color-warning);
      background-color: rgba(var(--color-warning-rgb), 0.1);
    }

    &.error {
      color: var(--color-error);
      background-color: rgba(var(--color-error-rgb), 0.1);
    }

    .length-text {
      font-weight: 500;
    }

    .size-text {
      opacity: 0.8;
      font-size: 10px;
    }

    .error-text {
      color: var(--color-error);
      font-weight: 600;
      font-size: 10px;
      text-transform: uppercase;
    }
  }
}

.send-button {
  height: 36px;
  padding: 0 16px;
  border-radius: 6px;
  font-weight: 500;
  background-color: var(--color-primary);
  border-color: var(--color-primary);
  color: white;
  transition: all 0.2s ease;

  &:hover:not(:disabled) {
    background-color: var(--color-primary-hover);
    border-color: var(--color-primary-hover);
    transform: translateY(-1px);
  }

  &:active:not(:disabled) {
    transform: translateY(0);
  }

  &:disabled {
    opacity: 0.5;
    background-color: var(--bg-tertiary);
    border-color: var(--border-primary);
    color: var(--text-disabled);
    cursor: not-allowed;
  }
}

// Terminal主题特殊样式
[data-theme='terminal'] {
  .chat-input {
    background-color: var(--terminal-bg);
    border-top-color: var(--terminal-border);
  }

  .input-area {
    background-color: var(--terminal-bg);
  }

  .message-input {
    :deep(.n-input) {
      background-color: var(--terminal-bg-secondary) !important;
      border-color: var(--terminal-border) !important;
      
      .n-input__input-el,
      .n-input__textarea-el {
        color: var(--terminal-text) !important;
        font-family: var(--font-mono) !important;
        
        &::placeholder {
          color: var(--terminal-text-tertiary) !important;
        }
      }

      &:hover {
        border-color: var(--terminal-border-active) !important;
        background-color: var(--terminal-bg-tertiary) !important;
      }

      &.n-input--focus {
        border-color: var(--pixel-green) !important;
        box-shadow: 0 0 0 2px rgba(0, 255, 65, 0.1) !important;
      }
    }
  }

  .send-button {
    background-color: var(--pixel-green);
    border-color: var(--pixel-green);
    color: var(--terminal-bg);
    font-family: var(--font-display);
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;

    &:hover:not(:disabled) {
      background-color: var(--pixel-cyan);
      border-color: var(--pixel-cyan);
      box-shadow: var(--neon-glow-cyan);
    }
  }

  .toolbar {
    .attachment-button,
    .image-button {
      color: var(--terminal-text-secondary);
      
      &:hover {
        background-color: var(--terminal-surface);
        color: var(--pixel-green);
      }
    }
  }
}

// 响应式设计
@media (max-width: 768px) {
  .input-area {
    padding: 8px 12px;
    gap: 8px;
  }

  .toolbar {
    .attachment-button,
    .image-button {
      width: 32px;
      height: 32px;
    }
  }

  .send-button {
    height: 32px;
    padding: 0 12px;
    font-size: 13px;
  }

  .message-input {
    :deep(.n-input) {
      .n-input__input-el,
      .n-input__textarea-el {
        font-size: 13px !important;
        padding: 6px 10px !important;
      }
    }

    :deep(.n-input__textarea-el) {
      min-height: 32px !important;
      max-height: 120px !important;
    }
  }
}

// 暗色主题特殊处理
@media (prefers-color-scheme: dark) {
  .message-input {
    :deep(.n-input) {
      // 确保在系统暗色模式下也有正确的样式
      background-color: var(--bg-secondary) !important;
      
      .n-input__input-el,
      .n-input__textarea-el {
        color: var(--text-primary) !important;
        
        &::placeholder {
          color: var(--text-tertiary) !important;
        }
      }
    }
  }
}
</style>
