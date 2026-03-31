<template>
  <v-app>
    <v-main>
      <v-container class="py-10" max-width="600">
        <v-card elevation="3" rounded="lg" class="mb-6">
          <v-card-title class="text-h5 pa-6">Send Test Message</v-card-title>
          <v-card-text class="px-6">
            <v-text-field
              v-model="form.sender_id"
              label="Sender ID"
              variant="outlined"
              class="mb-3"
            />
            <v-textarea v-model="form.text" label="Message" variant="outlined" rows="3" />
            <v-alert
              v-if="successMessage"
              type="success"
              class="mt-3"
              closable
              @click:close="successMessage = ''"
            >
              {{ successMessage }}
            </v-alert>
            <v-alert
              v-if="errorMessage"
              type="error"
              class="mt-3"
              closable
              @click:close="errorMessage = ''"
            >
              {{ errorMessage }}
            </v-alert>
          </v-card-text>
          <v-card-actions class="px-6 pb-6">
            <v-btn color="primary" size="large" :loading="loading" @click="sendMessage" block>
              Send Message
            </v-btn>
          </v-card-actions>
        </v-card>

        <v-card elevation="3" rounded="lg">
          <v-card-title class="pa-6">
            Live Socket Events
            <v-chip color="green" size="small" class="ml-2" v-if="connected">Connected</v-chip>
            <v-chip color="red" size="small" class="ml-2" v-else>Disconnected</v-chip>
          </v-card-title>
          <v-card-text>
            <v-list v-if="socketMessages.length">
              <v-list-item v-for="(msg, i) in socketMessages" :key="i" :subtitle="msg.timestamp">
                <template #title>
                  <span class="font-weight-bold">{{ msg.sender_id }}:</span>
                  {{ msg.text }}
                </template>
              </v-list-item>
            </v-list>
            <p v-else class="text-medium-emphasis pa-2">No socket events yet...</p>
          </v-card-text>
        </v-card>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref, onMounted } from 'vue'
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
    console.log('📩 Socket event received:', data)
    socketMessages.value.unshift({
      text: data.text,
      sender_id: data.sender_id,
      timestamp: new Date(data.timestamp).toLocaleString(),
    })
  })
  socket.on('disconnect', () => {
    connected.value = false
    console.log('❌ Socket disconnected')
  })
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
