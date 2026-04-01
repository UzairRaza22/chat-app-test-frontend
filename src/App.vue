<template>
  <v-app>
    <v-main>
      <div style="padding: 20px">
        <h2>Socket Test App</h2>

        <p>Status: {{ connected ? '🟢 Connected' : '🔴 Disconnected' }}</p>

        <input v-model="form.text" placeholder="Message" />
        <input v-model="form.sender_id" placeholder="Sender ID" />

        <button @click="sendMessage" :disabled="loading">Send</button>

        <p style="color: green">{{ successMessage }}</p>
        <p style="color: red">{{ errorMessage }}</p>

        <hr />

        <h3>Socket Messages</h3>
        <ul>
          <li v-for="(msg, index) in socketMessages" :key="index">
            {{ msg.text }} - {{ msg.sender_id }} ({{ msg.timestamp }})
          </li>
        </ul>
      </div>
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

const socket = io('http://178.104.58.236', {
  path: '/test-socket/socket.io',
  transports: ['polling', 'websocket'],
})

onMounted(() => {
  socket.on('connect', () => {
    connected.value = true
    console.log('✅ Socket connected:', socket.id)
  })

  socket.on('test_message_received', (data) => {
    socketMessages.value.unshift({
      text: data.text,
      sender_id: data.sender_id,
      timestamp: new Date(data.timestamp).toLocaleString(),
    })
  })

  socket.on('disconnect', () => {
    connected.value = false
  })
})

onUnmounted(() => {
  socket.disconnect()
})

async function sendMessage() {
  if (!form.value.text || !form.value.sender_id) {
    errorMessage.value = 'Please fill in both fields.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  successMessage.value = ''
  try {
    await axios.post('http://178.104.58.236/test-backend/api/message', {
      text: form.value.text,
      sender_id: form.value.sender_id,
    })
    successMessage.value = 'Message sent! Watch the socket event below.'
    form.value.text = ''
  } catch (err) {
    errorMessage.value = err?.response?.data?.message || 'Something went wrong.'
  } finally {
    loading.value = false
  }
}
</script>
