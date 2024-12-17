<template>
  <div class="chat-container">
    <div class="top">
      <div>JJAI Chat</div>
    </div>
    <div class="chatBox" v-if="!messages.length">
      <div class="boxImg"><img src="@/assets/logo.png" class="robotImg" /></div>
    </div>
    <!-- 中部内容区 -->
    <div class="chatInfor" v-if="!messages.length">
      <div class="chatContent">
        JJChat is an artificial-intelligence catbot developed by JJAI
      </div>
      <div class="chatContent">JJAI launched in Noveber 2024</div>
      <div class="chatContent">
        JJChat is an artificial-intelligence Chatbot developed by JJAI
      </div>
      <div class="description">This is example that what can I do for you</div>
    </div>
    <div v-else>
      <!-- 显示聊天记录 -->
      <div class="boxHeight">
        <div class="chat-box">
          <div class="messages">
            <div
              v-for="(message, index) in messages"
              :key="index"
              class="message-wrapper"
              :class="message.role === 'user' ? 'user-message' : 'ai-message'"
            >
              <div
                class="message"
                v-html="renderMessageContent(message.content)"
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="bottom">
      <div class="search">
        <div class="input-box">
          <textarea
            class="textVal"
            v-model="userInput"
            placeholder="请输入内容开始聊天"
            @keyup.enter="sendMessage"
          ></textarea>
        </div>
      </div>
      <img src="@/assets/send.png" class="robotImg" @click="sendMessage" />
    </div>
  </div>
</template>

<script setup lang="ts">
import Clipboard from 'clipboard'
import { ref, nextTick, watch, onMounted, onUnmounted } from 'vue'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'

const flag = ref(false)
const messages = ref([])
const userInput = ref('')
let clipboard = null

let isStreaming = false // 是否正在流式接收数据
let streamingMessage = '' // 当前正在接收的消息

var ws = new WebSocket('ws://localhost:3004')

ws.onopen = function (e) {
  console.log('websocket连接成功', e)
}

onMounted(() => {
  document.addEventListener('DOMContentLoaded', function () {
    // 初始化 Clipboard.js
    new Clipboard('.copy-btn')
  })
  nextTick(() => {
    clipboard = new Clipboard('.copy-btn')
    clipboard.on('success', e => {
      console.log('复制成功')
    })
    clipboard.on('error', e => {
      console.log('复制失败')
    })
  })
})

onUnmounted(() => {
  if (clipboard) {
    clipboard.destroy()
  }
})

const md = new MarkdownIt({
  highlight: (code, lang) => {
    const linesLength = code.split(/\n/).length - 1
    const codeIndex =
      parseInt(Date.now()) + Math.floor(Math.random() * 10000000)

    let html = `<button class="copy-btn" style="position: absolute;top: 0;right: 0;" type="button" data-clipboard-action="copy" data-clipboard-target="#copy${codeIndex}">复制</button>`

    if (lang && hljs.getLanguage(lang)) {
      if (linesLength) {
        html += '<b class="name">' + lang + '</b>'
      }

      const highlightedCode = hljs.highlight(code, { language: lang }).value
      return `<pre class="hljs" style="overflow-x:auto;position:relative"><code style="display:flex">${html}</code>${highlightedCode}</pre><textarea style="position: absolute;top: -9999px;left: -9999px;z-index: -9999;" id="copy${codeIndex}">${code.replace(
        /<\/textarea>/g,
        '&lt;/textarea>'
      )}</textarea>`
    }
    return ''
  }
})

// 渲染消息内容（支持 Markdown 和普通文本）
const renderMessageContent = content => {
  return md.render(content)
}

ws.onmessage = function (e) {
  const answer = JSON.parse(e.data)
  flag.value = true

  console.log('收到数据', answer)

  // 如果是流式输出，逐步更新
  if (answer == '{"reply":"","isStreaming":true}') {
    isStreaming = true
    streamingMessage = ''
    messages.value.push({ role: 'assistant', content: '' })
    console.log('流式第一次加载:', messages.value)
  } else if (isStreaming && answer.reply) {
    streamingMessage += answer.reply
    messages.value[messages.value.length - 1].content = streamingMessage
    console.log('最后的messages', messages.value)
  } else {
    isStreaming = false
  }
}

ws.onclose = function () {
  console.log('websocket已断开')
}

ws.onerror = function (error) {
  console.log('websocket发生错误' + error)
}

watch(
  messages,
  () => {
    nextTick(() => {
      document.querySelectorAll('pre code').forEach(el => {
        hljs.highlightElement(el)
      })
      // 重新初始化 Clipboard
      if (clipboard) {
        clipboard.destroy()
      }
      clipboard = new Clipboard('.copy-btn')
      clipboard.on('success', e => {
        console.log('复制成功')
      })
      clipboard.on('error', e => {
        console.log('复制失败')
      })
    })
  },
  { immediate: true },
  { deep: true }
)

// 发送用户消息
const sendMessage = () => {
  if (!userInput.value.trim()) return

  // 添加用户输入到消息列表
  messages.value.push({ role: 'user', content: userInput.value })

  // 通过 WebSocket 发送到后端
  ws.send(
    JSON.stringify({
      event: 'message',
      data: {
        content: userInput.value
      }
    })
  )

  userInput.value = '' // 清空输入框
}
</script>
<style scoped lang="less">
pre.hljs {
  padding: 12px 2px 12px 40px !important;
  border-radius: 5px !important;
  position: relative;
  font-size: 14px !important;
  line-height: 22px !important;
  overflow: hidden !important;

  code {
    display: block !important;
    margin: 10px 10px !important;
    overflow-x: auto !important;
  }
  .line-numbers-rows {
    position: absolute;
    pointer-events: none;
    top: 12px;
    bottom: 12px;
    left: 0;
    font-size: 100%;
    width: 40px;
    text-align: center;
    letter-spacing: -1px;
    border-right: 1px solid rgba(0, 0, 0, 0.66);
    user-select: none;
    counter-reset: linenumber;
    span {
      pointer-events: none;
      display: block;
      counter-increment: linenumber;
      &:before {
        content: counter(linenumber);
        color: #999;
        display: block;
        text-align: center;
      }
    }
  }
  b.name {
    position: absolute;
    top: 2px;
    right: 50px;
    z-index: 10;
    color: #999;
    pointer-events: none;
  }
  .copy-btn {
    position: absolute;
    top: 2px;
    right: 4px;
    z-index: 10;
    color: #333;
    cursor: pointer;
    background-color: #fff;
    border: 0;
    border-radius: 2px;
  }
}

.chat-container {
  padding-bottom: 30px;
  .boxHeight {
    height: 78vh;
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
          overflow-y: auto;
          max-width: 70%;
          padding: 5px 16px;
          border-radius: 18px;
          font-size: 20px;
          line-height: 1.5;
          white-space: pre-wrap;
          word-wrap: break-word;
          box-shadow: 0 1px 2px rgba(0, 0, 0, 0.15);
        }
      }

      .user-message {
        justify-content: flex-end;

        .message {
          background-color: #17c3ce;
          color: #ffffff;
          text-align: right;
          font-size: 14px;
          border-bottom-right-radius: 4px;
        }
        font-size: 20px;
      }

      .ai-message {
        justify-content: flex-start;

        .message {
          background-color: #f1f0f0;
          color: #333333;
          text-align: left;
          font-size: 14px;
          border-bottom-left-radius: 4px;
        }
      }
    }
  }
  .top {
    padding-top: 40px;
    padding-bottom: 20px;
    text-align: center;
    font-size: 24px;
    font-weight: 600;
    border-bottom: 1px solid #473f3f54;
    max-height: 4vh;
  }
  .boxImg {
    margin-top: 100px;
    text-align: center;
    margin-bottom: 120px;

    .robotImg {
      width: 160px;
      height: 160px;
    }
  }
  .chatInfor {
    padding: 0 10px;
    .chatContent {
      font-size: 14px;
      text-align: center;
      margin: 20px 0;
      padding: 30px 10px;
      background-color: #f5f5f5;
      color: #bbbbbb;
      border-radius: 20px;
    }
    .description {
      font-size: 14px;
      color: #bbbbbb;
      margin: 20px 0;
      text-align: center;
      padding: 30px 0px 30px 0;
    }
  }
  .bottom {
    // height: 4vh;
    display: flex;
    .search {
      // width: 550px;
      // margin-top: 50px;
      padding: 4px 10px 5px 10px;
      // margin-bottom: 50px;
      // margin-right: 5vw;
    }
    .robotImg {
      padding-top: 3px;
      // margin-top: 36px;
      width: 15vw;
      height: 15vw;
    }
    .input-box {
      background-color: #f5f5f5;
      width: 70vw;
      border-radius: 20px;
      // height: 40px;
      display: flex;
      align-items: center;
      gap: 8px;
      border-top: 1px solid #e5e5e5;
      padding: 4px 10px;
      .textVal {
        height: 20px;
        padding: 10px 0;
        font-size: 12px;
      }

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
        padding: 0 10px;
        font-size: 10px;
        border: none;
        resize: none;
        font-size: 14px;
        background-color: #f5f5f5;
        outline: none;
        // height: 30px;
      }
    }
  }
  :deep(textarea::-webkit-input-placeholder) {
    color: #9d9a9a;
    font-size: 14px !important;
  }
}
</style>
