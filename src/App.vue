<template>
  <v-app>
    <v-main>
      <v-container style="max-width: 600px; padding-top: 40px">
        <v-card elevation="3" rounded="lg" class="mb-4">
          <v-card-title>
            Socket Test App
            <v-chip :color="connected ? 'green' : 'red'" size="small" class="ml-2">
              {{ connected ? 'Connected' : 'Disconnected' }}
            </v-chip>
          </v-card-title>

          <v-card-text>
            <v-text-field
              v-model="form.sender_id"
              label="Your Name / Sender ID"
              variant="outlined"
              density="compact"
              class="mb-2"
            />
            <v-textarea
              v-model="form.text"
              label="Type a message..."
              variant="outlined"
              rows="2"
              @input="onTyping"
            />

            <v-alert
              v-if="successMessage"
              type="success"
              density="compact"
              class="mt-2"
              closable
              @click:close="successMessage = ''"
            >
              {{ successMessage }}
            </v-alert>
            <v-alert
              v-if="errorMessage"
              type="error"
              density="compact"
              class="mt-2"
              closable
              @click:close="errorMessage = ''"
            >
              {{ errorMessage }}
            </v-alert>
          </v-card-text>

          <v-card-actions class="px-4 pb-4">
            <v-btn color="primary" :loading="loading" @click="sendMessage" block>
              Send Message
            </v-btn>
          </v-card-actions>
        </v-card>

        <!-- Typing indicator -->
        <p v-if="typingText" style="color: gray; font-style: italic; margin-bottom: 8px">
          {{ typingText }}
        </p>

        <!-- Messages -->
        <v-card elevation="3" rounded="lg">
          <v-card-title>Live Messages</v-card-title>
          <v-card-text>
            <v-list v-if="socketMessages.length">
              <v-list-item
                v-for="(msg, index) in socketMessages"
                :key="index"
                :subtitle="msg.timestamp"
              >
                <template #title>
                  <span class="font-weight-bold">{{ msg.sender_id }}:</span>
                  {{ msg.text }}
                </template>
              </v-list-item>
            </v-list>
            <p v-else class="text-medium-emphasis">No messages yet...</p>
          </v-card-text>
        </v-card>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import axios from 'axios'
import { io } from 'socket.io-client'

const form = ref({ text: '', sender_id: '' })
const loading = ref(false)
const successMessage = ref('')
const errorMessage = ref('')
const socketMessages = ref([])
const connected = ref(false)
const typingText = ref('')

let typingTimer = null

// Connect to socket server
const socket = io('http://178.104.58.236', {
  path: '/test-socket/socket.io',
  transports: ['polling', 'websocket'],
})

onMounted(() => {
  socket.on('connect', () => {
    connected.value = true
    console.log('✅ Socket connected:', socket.id)
  })

  // Someone else is typing
  socket.on('user_typing', (data) => {
    if (data.isTyping) {
      typingText.value = `${data.sender_id} is typing...`
    } else {
      typingText.value = ''
    }
  })

  // Receive message sent directly via socket (instant)
  socket.on('message_received', (data) => {
    console.log('📩 Message received via socket:', data)
    socketMessages.value.unshift({
      text: data.text,
      sender_id: data.sender_id,
      timestamp: new Date(data.timestamp).toLocaleString(),
    })
  })

  // Also receive from MongoDB change stream watcher
  socket.on('test_message_received', (data) => {
    console.log('📩 Message from DB watcher:', data)
    // Avoid duplicates — only add if not already in list
    const exists = socketMessages.value.some(
      (m) => m.text === data.text && m.sender_id === data.sender_id,
    )
    if (!exists) {
      socketMessages.value.unshift({
        text: data.text,
        sender_id: data.sender_id,
        timestamp: new Date(data.timestamp).toLocaleString(),
      })
    }
  })

  socket.on('disconnect', () => {
    connected.value = false
    console.log('❌ Socket disconnected')
  })
})

onUnmounted(() => {
  socket.disconnect()
})

// Emit typing event while user types
function onTyping() {
  if (!form.value.sender_id) return

  socket.emit('typing', {
    sender_id: form.value.sender_id,
    isTyping: true,
  })

  // Stop typing after 1.5 seconds of no input
  clearTimeout(typingTimer)
  typingTimer = setTimeout(() => {
    socket.emit('typing', {
      sender_id: form.value.sender_id,
      isTyping: false,
    })
  }, 1500)
}

async function sendMessage() {
  if (!form.value.text || !form.value.sender_id) {
    errorMessage.value = 'Please fill in both fields.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  successMessage.value = ''

  // Stop typing indicator
  socket.emit('typing', { sender_id: form.value.sender_id, isTyping: false })
  clearTimeout(typingTimer)

  try {
    // 1. Emit directly via socket to all other connected clients (instant)
    socket.emit('send_message', {
      text: form.value.text,
      sender_id: form.value.sender_id,
    })

    // 2. Also save to MongoDB via Laravel (persistence)
    await axios.post('http://178.104.58.236/test-backend/api/message', {
      text: form.value.text,
      sender_id: form.value.sender_id,
    })

    // 3. Show sent message on sender's own screen too
    socketMessages.value.unshift({
      text: form.value.text,
      sender_id: form.value.sender_id + ' (you)',
      timestamp: new Date().toLocaleString(),
    })

    successMessage.value = 'Message sent!'
    form.value.text = ''
  } catch (err) {
    errorMessage.value = err?.response?.data?.message || 'Something went wrong.'
  } finally {
    loading.value = false
  }
}
</script>
