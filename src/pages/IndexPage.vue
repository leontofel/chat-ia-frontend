<template>
  <q-page class="chat-page">
    <div class="chat-shell">

      <!-- Header -->
      <q-card class="chat-header q-pa-md">
        <div class="row items-center no-wrap">
          <div class="col">
            <div class="text-h6 text-weight-bold">DevIA</div>
            <div class="text-caption opacity-80">
              Ollama via WebSocket
            </div>
          </div>

          <q-badge
            :color="wsStatus === 'open' ? 'positive' : (wsStatus === 'connecting' ? 'warning' : 'negative')"
            class="q-ml-sm"
            rounded
          >
            {{ wsStatusLabel }}
          </q-badge>
        </div>
      </q-card>

      <!-- Chat body -->
      <q-card class="chat-body">
        <q-scroll-area ref="scrollArea" class="chat-scroll">
          <div class="q-pa-md">

            <div v-if="chat.qa.length === 0" class="empty-state q-pa-xl">
              <div class="text-h6 text-weight-bold q-mb-sm">Start a conversation</div>
              <div class="text-body2 opacity-80">
                Ask something like “Create a Maven Spring Boot WebSocket API for Ollama”.
              </div>
              <q-chip class="q-mt-md" color="primary" text-color="white" icon="tips_and_updates">
                Tip: Press Enter to send • Shift+Enter for new line
              </q-chip>
            </div>

            <div v-for="(qa, idx) in chat.qa" :key="idx" class="q-mb-lg">

              <!-- User bubble -->
              <div class="row justify-end">
                <div class="bubble bubble-user">
                  <div class="bubble-text">{{ qa.question }}</div>
                </div>
              </div>

              <!-- Assistant bubble -->
              <div class="row justify-start q-mt-sm">
                <div class="bubble bubble-ai">
                  <div class="row items-center q-mb-xs">
                    <q-avatar size="24px" class="q-mr-sm" color="grey-9" text-color="white">
                      AI
                    </q-avatar>
                    <div class="text-caption opacity-70">Assistant</div>
                  </div>

                  <div class="bubble-text">
                    <!-- show streaming content only for last message being generated -->
                    <template v-if="isLoading && idx === chat.qa.length - 1">
                      <span v-if="message.length">{{ message }}</span>
                      <span v-else class="opacity-70">Thinking…</span>

                      <span class="cursor" aria-hidden="true">▍</span>
                    </template>

                    <template v-else>
                      <span v-if="qa.answer">{{ qa.answer }}</span>
                      <span v-else class="opacity-70">…</span>
                    </template>
                  </div>
                </div>
              </div>

            </div>
          </div>
        </q-scroll-area>
      </q-card>

      <!-- Composer (sticky) -->
      <q-card class="chat-composer q-pa-md">
        <q-form @submit.prevent="onSubmit" class="row items-end q-col-gutter-sm">
          <div class="col">
            <q-input
              v-model="input"
              type="textarea"
              autogrow
              :max-height="160"
              outlined
              rounded
              placeholder="Type your message…"
              :disable="wsStatus !== 'open'"
              @keydown.enter.exact.prevent="onSubmit"
              @keydown.enter.shift.exact.stop
            >
              <template #prepend>
                <q-icon name="chat_bubble_outline" class="opacity-70" />
              </template>
            </q-input>
          </div>

          <div class="col-auto">
            <q-btn
              color="primary"
              rounded
              unelevated
              icon="send"
              label="Send"
              :loading="isLoading"
              :disable="wsStatus !== 'open' || !input.trim()"
              @click="onSubmit"
            />
          </div>
        </q-form>

        <div class="row items-center q-mt-sm">
          <q-chip dense square class="q-mr-sm" icon="memory" color="grey-9" text-color="white">
            model: deepseek-r1:1.5b
          </q-chip>
          <div class="text-caption opacity-70">
            {{ wsStatus === 'open' ? 'Ready' : 'WebSocket not connected' }}
          </div>

          <q-space />

          <q-btn
            flat
            dense
            icon="delete_sweep"
            label="Clear"
            class="opacity-70"
            @click="clearChat"
          />
        </div>
      </q-card>

    </div>
  </q-page>
</template>

<script setup lang="ts">
import type { Chat } from 'src/types'
import { nextTick, onMounted, onBeforeUnmount, ref, computed } from 'vue'

const connection = ref<WebSocket | null>(null)
const wsStatus = ref<'connecting' | 'open' | 'closed'>('connecting')

const message = ref('')
const input = ref('')
const chat = ref<Chat>({ qa: [] })
const isLoading = ref(false)

// Quasar QScrollArea exposes methods; we’ll use its internal API safely
const scrollArea = ref()

const wsStatusLabel = computed(() => {
  if (wsStatus.value === 'open') return 'Connected'
  if (wsStatus.value === 'connecting') return 'Connecting'
  return 'Disconnected'
})

async function scrollToBottom() {
  await nextTick(() => {
    try {
      const sa = scrollArea.value
      // QScrollArea has setScrollPosition('vertical', value, duration)
      // We scroll to a very large number to reach bottom.
      sa?.setScrollPosition?.('vertical', 10_000_000, 200)
    } catch (e) {
      console.error(e)
    }
  })
}

function connectWs() {
  wsStatus.value = 'connecting'
  const ws = new WebSocket('ws://localhost:8080/ws/chat')
  connection.value = ws

  ws.onopen = () => {
    wsStatus.value = 'open'
  }

  ws.onclose = () => {
    wsStatus.value = 'closed'
    isLoading.value = false
  }

  ws.onerror = () => {
    wsStatus.value = 'closed'
    isLoading.value = false
  }

  ws.onmessage = async (event) => {
    if (!event.data) return
    const json = JSON.parse(event.data)

    if (json.type !== 'done') {
      message.value += json.content ?? ''
      await scrollToBottom()
      return
    }

    // done
    const last = chat.value.qa[chat.value.qa.length - 1]
    if (last) last.answer = message.value

    isLoading.value = false
    message.value = ''
    input.value = ''
    await scrollToBottom()
  }
}

onMounted(() => {
  connectWs()
})

onBeforeUnmount(() => {
  connection.value?.close()
})

async function onSubmit() {
  const prompt = input.value.trim()
  if (!prompt || isLoading.value) return
  if (!connection.value || wsStatus.value !== 'open') return

  try {
    isLoading.value = true
    message.value = ''

    chat.value.qa.push({ question: prompt, answer: '' })
    await scrollToBottom()

    // IMPORTANT: escape quotes/newlines safely by sending JSON stringified object
    const payload = {
      model: 'deepseek-r1:1.5b',
      prompt,
      stream: true
    }

    connection.value.send(JSON.stringify(payload))
  } catch (e) {
    console.error(e)
    isLoading.value = false
  }
}

function clearChat() {
  chat.value.qa = []
  message.value = ''
  input.value = ''
}
</script>

<style scoped>
.chat-page {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  padding: 24px 12px;
  background:
    radial-gradient(1200px 600px at 20% 10%, rgba(33, 150, 243, 0.18), transparent 55%),
    radial-gradient(900px 500px at 80% 30%, rgba(156, 39, 176, 0.16), transparent 55%),
    radial-gradient(900px 600px at 50% 90%, rgba(0, 150, 136, 0.14), transparent 60%);
}

.chat-shell {
  width: 100%;
  max-width: 980px;
  display: grid;
  grid-template-rows: auto 1fr auto;
  gap: 12px;
}

.chat-header {
  border-radius: 18px;
  backdrop-filter: blur(10px);
}

.chat-body {
  border-radius: 18px;
  overflow: hidden;
}

.chat-scroll {
  height: calc(100vh - 220px);
}

.chat-composer {
  border-radius: 18px;
  position: sticky;
  bottom: 12px;
}

.empty-state {
  border: 1px dashed rgba(255, 255, 255, 0.18);
  border-radius: 18px;
}

.bubble {
  max-width: min(720px, 92%);
  padding: 12px 14px;
  border-radius: 16px;
  line-height: 1.4;
  white-space: pre-wrap;
  word-break: break-word;
}

.bubble-user {
  background: rgba(33, 150, 243, 0.18);
  border: 1px solid rgba(33, 150, 243, 0.28);
}

.bubble-ai {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.10);
}

.bubble-text {
  font-size: 0.95rem;
}

.cursor {
  display: inline-block;
  margin-left: 2px;
  animation: blink 1s steps(1) infinite;
  opacity: 0.8;
}

@keyframes blink {
  50% { opacity: 0.15; }
}
</style>
