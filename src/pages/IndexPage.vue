<template>
  <q-page class="chat-page">
    <div class="chat-shell">

      <!-- Header -->
      <q-card class="chat-header q-pa-md">
        <div class="row items-center no-wrap">
          <div class="col">
            <div class="text-h6 text-weight-bold">DevIA</div>
            <div class="text-caption opacity-80">
              session: <span class="text-mono">{{ sessionId }}</span>
            </div>
          </div>

          <q-badge
            :color="wsStatus === 'open' ? 'positive' : (wsStatus === 'connecting' ? 'warning' : 'negative')"
            rounded
          >
            {{ wsStatusLabel }}
          </q-badge>
        </div>
      </q-card>

      <!-- Chat -->
      <q-card class="chat-body">
        <q-scroll-area ref="scrollArea" class="chat-scroll">
          <div class="q-pa-md">
            <div v-if="chat.qa.length === 0" class="empty-state q-pa-xl">
              <div class="text-h6 text-weight-bold q-mb-sm">Start a conversation</div>
              <div class="text-body2 opacity-80">
                Markdown supported: code blocks, lists, bold, etc.
              </div>
            </div>

            <div v-for="(qa, idx) in chat.qa" :key="idx" class="q-mb-lg">
              <!-- user -->
              <div class="row justify-end">
                <div class="bubble bubble-user">
                  <div class="bubble-text">{{ qa.question }}</div>
                </div>
              </div>

              <!-- assistant -->
              <div class="row justify-start q-mt-sm">
                <div class="bubble bubble-ai">
                  <div class="row items-center q-mb-xs">
                    <q-avatar size="24px" class="q-mr-sm" color="grey-9" text-color="white">
                      AI
                    </q-avatar>
                    <div class="text-caption opacity-70">Assistant</div>
                    <q-space />
                    <q-chip v-if="qa.cancelled" dense color="warning" text-color="black">cancelled</q-chip>
                  </div>

                  <div class="bubble-text">
                    <!-- streaming content for last item -->
                    <template v-if="isLoading && idx === chat.qa.length - 1">
                      <div v-if="message.length" v-html="renderMarkdown(message)" />
                      <div v-else class="typing">
                        <span class="dot" />
                        <span class="dot" />
                        <span class="dot" />
                      </div>
                    </template>

                    <template v-else>
                      <div v-if="qa.answer" v-html="renderMarkdown(qa.answer)" />
                      <span v-else class="opacity-70">…</span>
                    </template>
                  </div>
                </div>
              </div>
            </div>

          </div>
        </q-scroll-area>
      </q-card>

      <!-- Composer -->
      <q-card class="chat-composer q-pa-md">
        <q-form @submit.prevent="onSubmit" class="row items-end q-col-gutter-sm">
          <div class="col">
            <q-input
              v-model="input"
              type="textarea"
              autogrow
              outlined
              rounded
              placeholder="Type your message… (Markdown supported)"
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
              :disable="wsStatus !== 'open' || !input.trim() || isLoading"
              @click="onSubmit"
            />
          </div>

          <div class="col-auto">
            <q-btn
              v-if="isLoading"
              color="negative"
              rounded
              flat
              icon="stop_circle"
              label="Stop"
              @click="stopGenerating"
            />
          </div>
        </q-form>

        <div class="row items-center q-mt-sm">
          <q-chip dense square class="q-mr-sm" icon="memory" color="grey-9" text-color="white">
            model: {{ model }}
          </q-chip>

          <div class="text-caption opacity-70">
            {{ wsStatus === 'open' ? 'Ready' : 'WebSocket not connected' }}
          </div>

          <q-space />

          <q-btn flat dense icon="delete_sweep" label="Clear" class="opacity-70" @click="clearChat" />
        </div>
      </q-card>

    </div>
  </q-page>
</template>

<script setup lang="ts">
import { nextTick, onMounted, onBeforeUnmount, ref, computed, watch } from 'vue'
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'
import DOMPurify from 'dompurify'

// Extend your Chat type to allow cancelled flag
type QA = { question: string; answer: string; cancelled?: boolean }
type Chat = { qa: QA[] }

const connection = ref<WebSocket | null>(null)
const wsStatus = ref<'connecting' | 'open' | 'closed'>('connecting')

const model = ref('gemma:2b')
const message = ref('')
const input = ref('')
const chat = ref<Chat>({ qa: [] })
const isLoading = ref(false)

const scrollArea = ref()

// --- Session + local history ---
const sessionId = ref<string>('')

function getOrCreateSessionId() {
  const key = 'devia_session_id'
  const existing = localStorage.getItem(key)
  if (existing) return existing
  const fresh = (crypto?.randomUUID?.() ?? String(Date.now()))
  localStorage.setItem(key, fresh)
  return fresh
}

function historyKey() {
  return `devia_history_${sessionId.value}`
}

function loadHistory() {
  try {
    const raw = localStorage.getItem(historyKey())
    if (!raw) return
    const parsed = JSON.parse(raw)
    if (parsed?.qa?.length) chat.value = parsed
  } catch { /* empty */ }
}

function saveHistory() {
  try {
    localStorage.setItem(historyKey(), JSON.stringify(chat.value))
  } catch {
    //
  }
}

watch(chat, saveHistory, { deep: true })

// --- Markdown renderer ---
const md: MarkdownIt = new MarkdownIt({
  linkify: true,
  breaks: true,
  highlight: (code, lang) => {
    try {
      if (lang && hljs.getLanguage(lang)) {
        return `<pre class="hljs"><code>${hljs.highlight(code, { language: lang }).value}</code></pre>`
      }
      return `<pre class="hljs"><code>${hljs.highlightAuto(code).value}</code></pre>`
    } catch {
      return `<pre class="hljs"><code>${md.utils.escapeHtml(code)}</code></pre>`
    }
  }
})

function renderMarkdown(text: string) {
  const unsafe = md.render(text || '')
  return DOMPurify.sanitize(unsafe)
}

// --- UI helpers ---
const wsStatusLabel = computed(() => {
  if (wsStatus.value === 'open') return 'Connected'
  if (wsStatus.value === 'connecting') return 'Connecting'
  return 'Disconnected'
})

async function scrollToBottom() {
  await nextTick(() => {
    try {
      scrollArea.value?.setScrollPosition?.('vertical', 10_000_000, 200)
    } catch {
      //
    }
  })
}

function makeWsUrl(path = "/ws/chat") {
  const proto = window.location.protocol === "https:" ? "wss:" : "ws:"
  return `${proto}//${window.location.host}${path}`
}

// --- WebSocket ---
function connectWs() {
  wsStatus.value = 'connecting'
  const ws = new WebSocket(makeWsUrl("/ws/chat"))
  connection.value = ws

  ws.onopen = () => (wsStatus.value = 'open')
  ws.onclose = () => { wsStatus.value = 'closed'; isLoading.value = false }
  ws.onerror = () => { wsStatus.value = 'closed'; isLoading.value = false }

  ws.onmessage = async (event) => {
    if (!event.data) return
    const json = JSON.parse(event.data)

    if (json.type === 'delta') {
      message.value += json.content ?? ''
      await scrollToBottom()
      return
    }

    if (json.type === 'cancelled') {
      const last = chat.value.qa[chat.value.qa.length - 1]
      if (last) {
        last.answer = message.value
        last.cancelled = true
      }
      isLoading.value = false
      message.value = ''
      await scrollToBottom()
      return
    }

    if (json.type === 'done') {
      const last = chat.value.qa[chat.value.qa.length - 1]
      if (last) last.answer = message.value
      isLoading.value = false
      message.value = ''
      input.value = ''
      await scrollToBottom()
      return
    }

    if (json.type === 'error') {
      const last = chat.value.qa[chat.value.qa.length - 1]
      if (last) last.answer = `**Error:** ${json.message ?? 'unknown'}`
      isLoading.value = false
      message.value = ''
      await scrollToBottom()
    }
  }
}

onMounted(async () => {
  sessionId.value = getOrCreateSessionId()
  loadHistory()
  connectWs()
  await scrollToBottom()
})

onBeforeUnmount(() => connection.value?.close())

async function onSubmit() {
  const prompt = input.value.trim()
  if (!prompt || isLoading.value) return
  if (!connection.value || wsStatus.value !== 'open') return

  isLoading.value = true
  message.value = ''

  chat.value.qa.push({ question: prompt, answer: '' })
  await scrollToBottom()

  // Send chat request with sessionId
  connection.value.send(JSON.stringify({
    type: 'chat',
    sessionId: sessionId.value,
    model: model.value,
    prompt
  }))
}

function stopGenerating() {
  if (!connection.value || wsStatus.value !== 'open') return
  connection.value.send(JSON.stringify({ type: 'cancel' }))
}

function clearChat() {
  chat.value = { qa: [] }
  message.value = ''
  input.value = ''
  saveHistory()
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

.chat-header, .chat-body, .chat-composer {
  border-radius: 18px;
}

.chat-scroll {
  height: calc(100vh - 220px);
}

.empty-state {
  border: 1px dashed rgba(255, 255, 255, 0.18);
  border-radius: 18px;
}

.bubble {
  max-width: min(760px, 92%);
  padding: 12px 14px;
  border-radius: 16px;
  line-height: 1.5;
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

.text-mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
}

/* Typing dots */
.typing {
  display: inline-flex;
  gap: 6px;
  align-items: center;
  padding: 4px 0;
}
.dot {
  width: 7px;
  height: 7px;
  border-radius: 999px;
  background: rgba(255,255,255,0.65);
  animation: bounce 1.2s infinite ease-in-out;
}
.dot:nth-child(2) { animation-delay: .15s; }
.dot:nth-child(3) { animation-delay: .3s; }

@keyframes bounce {
  0%, 80%, 100% { transform: translateY(0); opacity: .5; }
  40% { transform: translateY(-4px); opacity: 1; }
}

/* Make code blocks look good */
:deep(pre.hljs) {
  padding: 12px;
  border-radius: 12px;
  overflow: auto;
  background: rgba(0,0,0,0.35);
  border: 1px solid rgba(255,255,255,0.08);
}
:deep(code) {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
}
</style>
