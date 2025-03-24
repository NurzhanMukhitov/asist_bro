<template>
  <div class="container">
    <div class="messages-container">
      <div v-for="message in messages" :key="message.id" 
           :class="['message', message.isUser ? 'message-user' : 'message-bot']">
        {{ message.text }}
        <div v-if="message.audioUrl" class="audio-player">
          <audio controls :src="message.audioUrl"></audio>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <input v-model="newMessage" 
             @keyup.enter="sendMessage"
             class="input-field" 
             placeholder="Введите сообщение..." />
      
      <button class="button" @click="startVoiceRecording" v-if="!isRecording">
        🎤
      </button>
      
      <button class="button recording" @click="stopVoiceRecording" v-else>
        ⏹️
      </button>
      
      <button class="button" @click="sendMessage" :disabled="!newMessage && !isRecording">
        ➤
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const messages = ref([])
const newMessage = ref('')
const isRecording = ref(false)
const mediaRecorder = ref(null)
const audioChunks = ref([])
const tg = window.Telegram.WebApp

onMounted(() => {
  // Инициализация Telegram Web App
  tg.ready()
  
  // Настройка цветовой схемы
  document.body.style.backgroundColor = tg.backgroundColor
  document.body.style.color = tg.textColor
  
  // Добавляем приветственное сообщение
  messages.value.push({
    id: Date.now(),
    text: 'Привет! Я ваш голосовой ассистент. Чем могу помочь?',
    isUser: false
  })
})

async function sendMessage() {
  if (!newMessage.value && !isRecording.value) return
  
  const messageText = newMessage.value
  
  // Добавляем сообщение пользователя
  messages.value.push({
    id: Date.now(),
    text: messageText,
    isUser: true
  })
  
  // Очищаем поле ввода
  newMessage.value = ''
  
  try {
    // Отправляем сообщение боту через Telegram Web App
    tg.sendData(JSON.stringify({
      type: 'message',
      text: messageText
    }))
    
    // В реальном приложении здесь нужно дождаться ответа от бота
    // Сейчас просто имитируем ответ
    setTimeout(() => {
      messages.value.push({
        id: Date.now(),
        text: 'Получил ваше сообщение! Скоро отвечу.',
        isUser: false
      })
    }, 1000)
  } catch (error) {
    console.error('Ошибка при отправке сообщения:', error)
    messages.value.push({
      id: Date.now(),
      text: 'Произошла ошибка при отправке сообщения',
      isUser: false
    })
  }
}

async function startVoiceRecording() {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    mediaRecorder.value = new MediaRecorder(stream)
    audioChunks.value = []
    
    mediaRecorder.value.ondataavailable = (event) => {
      audioChunks.value.push(event.data)
    }
    
    mediaRecorder.value.onstop = async () => {
      const audioBlob = new Blob(audioChunks.value, { type: 'audio/ogg; codecs=opus' })
      
      try {
        // Отправляем голосовое сообщение боту через Telegram Web App
        tg.sendData(JSON.stringify({
          type: 'voice',
          audio: await blobToBase64(audioBlob)
        }))
        
        // В реальном приложении здесь нужно дождаться ответа от бота
        // Сейчас просто имитируем ответ
        setTimeout(() => {
          messages.value.push({
            id: Date.now(),
            text: 'Получил ваше голосовое сообщение! Скоро отвечу.',
            isUser: false
          })
        }, 1000)
      } catch (error) {
        console.error('Ошибка при отправке голосового сообщения:', error)
        messages.value.push({
          id: Date.now(),
          text: 'Произошла ошибка при отправке голосового сообщения',
          isUser: false
        })
      }
    }
    
    mediaRecorder.value.start()
    isRecording.value = true
  } catch (error) {
    console.error('Ошибка при записи голоса:', error)
    messages.value.push({
      id: Date.now(),
      text: 'Не удалось получить доступ к микрофону',
      isUser: false
    })
  }
}

function stopVoiceRecording() {
  if (mediaRecorder.value && mediaRecorder.value.state === 'recording') {
    mediaRecorder.value.stop()
    isRecording.value = false
  }
}

function blobToBase64(blob) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onloadend = () => resolve(reader.result.split(',')[1])
    reader.onerror = reject
    reader.readAsDataURL(blob)
  })
}
</script>

<style>
.recording {
  background-color: var(--tg-theme-destructive-text-color, #ff0000) !important;
}

.audio-player {
  margin-top: 0.5rem;
}

.audio-player audio {
  width: 100%;
  height: 30px;
}
</style>
