<template>
  <div class="main">
    <div class="dialog-content">
      <BubbleList :list="state.messages" max-height="100%" maxWidth="1000px">
        <!-- 自定义头像 -->
        <template #avatar="{ item }">
          <div class="avatar-wrapper">
            <img :src="item.role === 'ai' ? avartAi : avatar" alt="avatar" />
          </div>
        </template>

        <!-- 自定义头部 -->
        <template #header="{ item }">
          <div class="header-wrapper">
            <div class="header-name">
              {{ item.role === "ai" ? "Element Plus X 🍧" : "🧁 用户" }}
            </div>
          </div>
        </template>

        <!-- 自定义气泡内容 -->
        <template #content="{ item }">
          <div class="content-wrapper">
            <div class="content-text" style="white-space: pre-wrap" v-html="item.content"></div>
          </div>
        </template>

        <!-- 自定义底部 -->
        <template #footer="{ item }">
          <div class="footer-wrapper">
            <div class="footer-container">
              <el-button type="info" :icon="Refresh" size="small" circle />
              <el-button type="success" :icon="Search" size="small" circle />
              <el-button type="warning" :icon="Star" size="small" circle />
              <el-button color="#626aef" :icon="DocumentCopy" size="small" circle />
            </div>
            <div class="footer-time">
              {{ item.date }}
            </div>
          </div>
        </template>

        <!-- 自定义 loading -->
        <template #loading="{ item }">
          <div class="loading-container">
            <span>#{{ item.role }}-{{ item.key }}：</span>
            <span>我</span>
            <span>是</span>
            <span>自</span>
            <span>定</span>
            <span>义</span>
            <span>加</span>
            <span>载</span>
            <span>内</span>
            <span>容</span>
            <span>哦</span>
            <span>~</span>
          </div>
        </template>
      </BubbleList>
    </div>
    <div class="text-content">
      <EditorSender
        :custom-style="{ maxHeight: '240px' }"
        placeholder="请输入内容"
        clearable
        :loading="state.loading"
        @cancel="handleCancel"
        @submit="handleSubmit"
        style="
          background-image: linear-gradient(to left, #7fffaa 0%, #00ffff 100%);
          border-radius: 8px;
        "
      />
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, reactive } from "vue"
import { BubbleList, Sender, EditorSender } from "vue-element-plus-x"
const list = [
  {
    content: "Hello, Element Plus X",
    role: "user",
  },
]
const state = reactive({
  loading: false,
  messages: [],
  eventBuffer: "",
})

// import { DocumentCopy, Refresh, Search, Star } from '@element-plus/icons-vue';

type listType = BubbleListItemProps & {
  key: number
  role: "user" | "ai"
}

generateFakeItems()
const avatar = ref("https://avatars.githubusercontent.com/u/76239030?v=4")
const avartAi = ref("https://cube.elemecdn.com/0/88/03b0d39583f48206768a7534e55bcpng.png")
const switchValue = ref(false)
const loading = ref(false)
function formatDateTime(date = new Date()) {
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, "0") // 月份从0开始，所以需要加1
  const day = String(date.getDate()).padStart(2, "0")
  const hours = String(date.getHours()).padStart(2, "0")
  const minutes = String(date.getMinutes()).padStart(2, "0")
  const seconds = String(date.getSeconds()).padStart(2, "0")

  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
}
function generateFakeItems(role = "ai", content = "你好，我是 Element Plus X"): listType[] {
  // 默认的ai 展示内容
  let key = state.messages.length + 1
  state.messages.push({
    key: key,
    role: role,
    placement: role === "ai" ? "start" : "end",
    noStyle: true, // 如果你不想用默认的气泡样式
    content: content,
    loading: false,
    date: formatDateTime(new Date()),
  })
  // for (let i = 0; i < count; i++) {
  //   const role = i % 2 === 0 ? "ai" : "user"
  //   const placement = role === "ai" ? "start" : "end"
  //   const key = i + 1
  //   messages.push({
  //     key,
  //     role,
  //     placement,
  //     noStyle: true, // 如果你不想用默认的气泡样式
  //   })
  // }
}

// 设置某个 item 的 loading
function setLoading(loading: boolean) {
  bubbleItems.value[bubbleItems.value.length - 1].loading = loading
  bubbleItems.value[bubbleItems.value.length - 2].loading = loading
}
// 发送请求
const handleSubmit = (val) => {
  console.log(val, "11")
  state.loading = true
  callSSEAPI(val.text)
  generateFakeItems("user", val.text)
}
const handleCancel = (val) => {
  state.loading = false
}
async function callSSEAPI(text) {
  const url = "https://api.siliconflow.cn/v1/messages"
  const headers = {
    Authorization: "Bearer sk-ycsqkhdpsksisdxfmomipclyeqnpiwtorfidrpljlsqcfsvk",
    "Content-Type": "application/json",
  }

  const body = JSON.stringify({
    model: "moonshotai/Kimi-K2-Instruct",
    messages: [{ role: "user", content: text }],
    stream: true,
    temperature: 0.7,
    max_tokens: 1024,
    tool_choice: {
      type: "auto",
      disable_parallel_tool_use: true,
    },
  })

  try {
    const response = await fetch(url, {
      method: "POST",
      headers: headers,
      body: body,
    })
    console.log(response, "response")

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const reader = response.body.getReader()

    const decoder = new TextDecoder("utf-8")
    console.log(decoder)
    let buffer = ""
    state.eventBuffer = ""
    state.messages[state.messages.length] = {
      key: state.messages.length + 1,
      role: "ai",
      placement: "ai" === "ai" ? "start" : "end",
      noStyle: true, // 如果你不想用默认的气泡样式
      content: state.eventBuffer,
      loading: false,
      date: formatDateTime(new Date()),
    }
    while (true) {
      // 作用：持续从 ReadableStream 中读取数据块。
      const { done, value } = await reader.read() // 返回一个 Promise，解析后得到 { done, value }。  value:当前读取到的数据块（Uint8Array 二进制数据）。
      if (done) break // 流是否结束（true 表示结束）。

      /*       使用 TextDecoder 将二进制数据 (Uint8Array) 解码为 UTF-8 字符串。
        注意：{ stream: true } 表示数据是分块的，可能包含不完整的字符（等待后续数据补齐）。 */
      buffer += decoder.decode(value, { stream: true }) // 将解码后的字符串追加到 buffer 变量中，用于累积未处理的流数据。
      // console.log(buffer, "buffer")

      // 处理缓冲区中的完整事件
      while (buffer.includes("\n\n")) {
        // 作用：查找 buffer 中是否存在 "\n\n" 字符串，即完整的事件。（SSE 事件以 \n\n 分隔）
        const eventEnd = buffer.indexOf("\n\n") // 找到第一个 \n\n 的位置，标记事件的结束。
        const eventStr = buffer.substring(0, eventEnd) // 截取从开头到 \n\n 的部分，得到一个完整的事件字符串。
        buffer = buffer.substring(eventEnd + 2) //更新 buffer，移除已处理的事件（+2 是跳过 \n\n）。

        // 解析SSE事件
        const event = {}

        /*         eventStr.split("\n")：
        将事件字符串按行拆分（SSE 的每行格式如 event:message 或 data:{"text":"hello"}）。
        line.indexOf(":")：
        找到冒号 : 的位置，分隔键和值。
        event[key] = value： */
        eventStr.split("\n").forEach((line) => {
          const colonIndex = line.indexOf(":")
          if (colonIndex > 0) {
            const key = line.substring(0, colonIndex).trim()
            const value = line.substring(colonIndex + 1).trim()
            event[key] = value
          }
        })

        // 处理有效事件
        if (event.data) {
          try {
            const data = JSON.parse(event.data)
            console.log(data, "data")

            // 这里处理不同类型的事件
            if (event.event === "message_start") {
              // 处理消息开始
            } else if (data.type === "content_block_delta") {
              console.log("Received event:", data)
              state.eventBuffer += data.delta.text

              // 更新最后一个 AI 消息的内容
              const lastIndex = state.messages.length - 1
              state.messages[lastIndex].content = state.eventBuffer
              state.messages[lastIndex].loading = false

              // 强制更新视图
              // Vue 3 的响应式系统可能需要显式地触发更新
              state.messages = [...state.messages]
              // 处理内容块
            }
            // 其他事件类型...
          } catch (e) {
            console.error("Error parsing event data:", e)
          }
        }
      }

      console.log(state.eventBuffer)
    }
    console.log("\n流处理完成")
    state.loading = false
  } catch (error) {
    console.error("请求失败:", error)
  }
}
</script>
<style scoped>
.main {
  width: 100%;
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  padding: 24px;
  .dialog-content {
    flex: 1;
    overflow: auto;
  }
  .text-content {
    flex-shrink: 0;
    margin-top: 20px;
  }
}
.avatar-wrapper {
  width: 40px;
  height: 40px;
  img {
    width: 100%;
    height: 100%;
    border-radius: 50%;
  }
}

.header-wrapper {
  .header-name {
    font-size: 14px;
    color: #979797;
  }
}

.content-wrapper {
  .content-text {
    font-size: 14px;
    color: #333;
    padding: 12px;
    background: linear-gradient(to right, #fdfcfb 0%, #ffd1ab 100%);
    border-radius: 15px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
}

.footer-wrapper {
  display: flex;
  align-items: center;
  gap: 10px;
  .footer-time {
    font-size: 12px;
    margin-top: 3px;
  }
}

.footer-container {
  :deep(.el-button + .el-button) {
    margin-left: 8px;
  }
}

.loading-container {
  font-size: 14px;
  color: #333;
  padding: 12px;
  background: linear-gradient(to right, #fdfcfb 0%, #ffd1ab 100%);
  border-radius: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.loading-container span {
  display: inline-block;
  margin-left: 8px;
}

@keyframes bounce {
  0%,
  100% {
    transform: translateY(5px);
  }
  50% {
    transform: translateY(-5px);
  }
}

.loading-container span:nth-child(4n) {
  animation: bounce 1.2s ease infinite;
}
.loading-container span:nth-child(4n + 1) {
  animation: bounce 1.2s ease infinite;
  animation-delay: 0.3s;
}
.loading-container span:nth-child(4n + 2) {
  animation: bounce 1.2s ease infinite;
  animation-delay: 0.6s;
}
.loading-container span:nth-child(4n + 3) {
  animation: bounce 1.2s ease infinite;
  animation-delay: 0.9s;
}

/deep/.el-bubble-no-style .el-bubble-content-wrapper .el-bubble-content {
  max-width: 90% !important;
}
.content-wrapper {
  max-width: 100% !important;
}
</style>
