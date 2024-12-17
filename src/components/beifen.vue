<template>
  <div class="chat-container">
    <div class="chat-box">
      <div class="messages">
        <!-- 显示聊天记录 -->
        <div
          v-for="(message, index) in messages"
          :key="index"
          class="message-wrapper"
          :class="message.role === 'user' ? 'user-message' : 'ai-message'"
        >
          <div class="message">
            <p>{{ message.content }}</p>
          </div>
        </div>
      </div>
    </div>
    <div class="input-box">
      <textarea
        v-model="userInput"
        placeholder="请输入您的问题..."
        @keyup.enter="sendMessage"
      ></textarea>
      <button @click="sendMessage">发送</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const messages = ref([])
const userInput = ref('')
const socket = new WebSocket('ws://localhost:3003')

// 监听服务端消息
socket.onmessage = event => {
  const data = JSON.parse(event.data)
  messages.value.push({ role: 'ai', content: data.reply })
}

// 发送用户消息
const sendMessage = () => {
  if (!userInput.value.trim()) return

  // 添加用户输入到消息列表
  messages.value.push({ role: 'user', content: userInput.value })
  console.log('userInput.value', userInput.value)
  // 通过 WebSocket 发送到后端
  socket.send(
    JSON.stringify({
      event: 'chat',
      data: {
        content: userInput.value
      }
    })
  )

  userInput.value = '' // 清空输入框
}
</script>
<style scoped lang="less">
.chat-container {
  height: 100vh;
  background-color: #f6f7f9;
  overflow: hidden;

  .chat-box {
    height: calc(100% - 60px);
    box-sizing: border-box;
    padding: 16px;
    overflow-y: auto;
    background-color: #ffffff;

    .messages {
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    .message-wrapper {
      display: flex;

      .message {
        max-width: 70%;
        padding: 5px 16px;
        border-radius: 18px;
        font-size: 14px;
        line-height: 1.5;
        white-space: pre-wrap;
        word-wrap: break-word;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.15);
      }
    }

    .user-message {
      justify-content: flex-end;

      .message {
        background-color: #0084ff;
        color: #ffffff;
        text-align: right;
        border-bottom-right-radius: 4px;
      }
    }

    .ai-message {
      justify-content: flex-start;

      .message {
        background-color: #f1f0f0;
        color: #333333;
        text-align: left;
        border-bottom-left-radius: 4px;
      }
    }
  }

  .input-box {
    height: 60px;
    display: flex;
    align-items: center;
    gap: 8px;
    background-color: #e5e5e5;
    border-top: 1px solid #e5e5e5;
    padding: 0 10px;

    button {
      padding: 5px 20px;
      background-color: #0084ff;
      color: #ffffff;
      border: none;
      border-radius: 10px;
      font-size: 14px;
      cursor: pointer;
      box-shadow: 0 2px 4px rgba(0, 132, 255, 0.3);
      transition: background-color 0.3s ease;
    }

    button:hover {
      background-color: #006bbf;
    }

    button:active {
      background-color: #0056a3;
    }

    textarea {
      flex: 1;
      padding: 10px;
      border: 1px solid #d5d5d5;
      border-radius: 15px;
      resize: none;
      font-size: 14px;
      background-color: #ffffff;
      box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.1);
      outline: none;
      height: 20px;
    }

    textarea:focus {
      border-color: #0084ff;
      box-shadow: inset 0 1px 4px rgba(0, 132, 255, 0.2);
    }
  }
}
</style>
