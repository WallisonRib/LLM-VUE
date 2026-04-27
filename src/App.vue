<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import {
  Bot,
  Download,
  Eraser,
  FileUp,
  MessageSquarePlus,
  Pencil,
  Send,
  Settings,
  Trash2,
  User,
} from 'lucide-vue-next'
import MarkdownIt from 'markdown-it'

const STORAGE_KEY = 'llm-vue.chats'
const SETTINGS_KEY = 'llm-vue.settings'

const provider = import.meta.env.VITE_LLM_PROVIDER || 'groq'
const apiKey = import.meta.env.VITE_GROQ_API_KEY || import.meta.env.VITE_LLM_API_KEY || ''
const baseUrl = import.meta.env.VITE_LLM_BASE_URL || 'https://api.groq.com/openai/v1'
const defaultModel = import.meta.env.VITE_GROQ_MODEL || import.meta.env.VITE_LLM_MODEL || 'llama-3.3-70b-versatile'
const defaultSystemPrompt =
  'Voce e um mentor sobre programacao. Explique conceitos com clareza, ajude a resolver problemas passo a passo e incentive boas praticas de desenvolvimento.'
const markdown = new MarkdownIt({
  breaks: true,
  linkify: true,
})

const now = () => new Date().toISOString()
const uid = () => crypto.randomUUID()

const buildChat = () => ({
  id: uid(),
  title: 'Novo chat',
  systemPrompt: defaultSystemPrompt,
  createdAt: now(),
  updatedAt: now(),
  messages: [],
})

const chats = ref([])
const activeChatId = ref('')
const prompt = ref('')
const isSending = ref(false)
const error = ref('')
const fileInput = ref(null)
const messagesList = ref(null)
const promptEditorOpen = ref(false)
const systemPromptDraft = ref('')
const settings = ref({
  model: defaultModel,
  temperature: 0.7,
})

const activeChat = computed(() => chats.value.find((chat) => chat.id === activeChatId.value))
const sortedChats = computed(() =>
  [...chats.value].sort((a, b) => new Date(b.updatedAt) - new Date(a.updatedAt)),
)
const canSend = computed(() => prompt.value.trim().length > 0 && !isSending.value)

function loadState() {
  const savedSettings = localStorage.getItem(SETTINGS_KEY)
  if (savedSettings) {
    settings.value = { ...settings.value, ...JSON.parse(savedSettings) }
  }

  const savedChats = localStorage.getItem(STORAGE_KEY)
  chats.value = savedChats ? JSON.parse(savedChats).map(normalizeChat) : [buildChat()]
  activeChatId.value = sortedChats.value[0]?.id || chats.value[0].id
}

function normalizeChat(chat) {
  return {
    id: chat.id || uid(),
    title: chat.title || 'Novo chat',
    systemPrompt: chat.systemPrompt || defaultSystemPrompt,
    createdAt: chat.createdAt || now(),
    updatedAt: chat.updatedAt || now(),
    messages: Array.isArray(chat.messages) ? chat.messages : [],
  }
}

function persistChats() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(chats.value))
}

function persistSettings() {
  localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings.value))
}

function createChat() {
  const chat = buildChat()
  chats.value.unshift(chat)
  activeChatId.value = chat.id
  prompt.value = ''
  error.value = ''
  openSystemPromptEditor()
}

function selectChat(id) {
  activeChatId.value = id
  prompt.value = ''
  error.value = ''
  scrollToBottom()
}

function renameActiveChat() {
  if (!activeChat.value) return
  const nextTitle = window.prompt('Nome do chat', activeChat.value.title)?.trim()
  if (!nextTitle) return
  activeChat.value.title = nextTitle
  activeChat.value.updatedAt = now()
}

function deleteChat(id) {
  const nextChats = chats.value.filter((chat) => chat.id !== id)

  if (chats.value.length === 1) {
    chats.value = [buildChat()]
    activeChatId.value = chats.value[0].id
    openSystemPromptEditor()
    return
  }

  chats.value = nextChats
  if (activeChatId.value === id) {
    activeChatId.value = sortedChats.value[0]?.id || chats.value[0]?.id
  }
}

function clearActiveChat() {
  if (!activeChat.value || activeChat.value.messages.length === 0) return
  activeChat.value.messages = []
  activeChat.value.title = 'Novo chat'
  activeChat.value.updatedAt = now()
}

function exportActiveChat() {
  if (!activeChat.value) return
  downloadJson(activeChat.value, `${safeFileName(activeChat.value.title)}.json`)
}

function exportAllChats() {
  downloadJson(
    {
      exportedAt: now(),
      provider,
      chats: chats.value,
    },
    `llm-vue-chats-${new Date().toISOString().slice(0, 10)}.json`,
  )
}

function downloadJson(data, fileName) {
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = fileName
  link.click()
  URL.revokeObjectURL(url)
}

function safeFileName(value) {
  return value
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/(^-|-$)/g, '') || 'chat'
}

function importChats(event) {
  const file = event.target.files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = () => {
    try {
      const data = JSON.parse(reader.result)
      const incoming = Array.isArray(data?.chats) ? data.chats : [data]
      const normalized = incoming
        .filter((chat) => Array.isArray(chat.messages))
        .map((chat) => ({
          id: chat.id || uid(),
          title: chat.title || 'Chat importado',
          systemPrompt: chat.systemPrompt || defaultSystemPrompt,
          createdAt: chat.createdAt || now(),
          updatedAt: chat.updatedAt || now(),
          messages: chat.messages,
        }))

      if (normalized.length === 0) {
        throw new Error('Arquivo sem chats validos.')
      }

      chats.value = [...normalized, ...chats.value]
      activeChatId.value = normalized[0].id
      error.value = ''
    } catch (importError) {
      error.value = importError.message || 'Nao foi possivel importar o arquivo.'
    } finally {
      event.target.value = ''
    }
  }
  reader.readAsText(file)
}

async function sendMessage() {
  if (!canSend.value || !activeChat.value) return

  error.value = ''

  if (!apiKey) {
    error.value = 'Informe VITE_GROQ_API_KEY no arquivo .env e reinicie o servidor.'
    return
  }

  const userMessage = {
    id: uid(),
    role: 'user',
    content: prompt.value.trim(),
    createdAt: now(),
  }
  const assistantMessage = {
    id: uid(),
    role: 'assistant',
    content: '',
    createdAt: now(),
  }

  activeChat.value.messages.push(userMessage, assistantMessage)
  if (activeChat.value.title === 'Novo chat') {
    activeChat.value.title = userMessage.content.slice(0, 42)
  }
  activeChat.value.updatedAt = now()
  prompt.value = ''
  isSending.value = true
  await scrollToBottom()

  try {
    const response = await fetch(`${baseUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: settings.value.model,
        temperature: Number(settings.value.temperature),
        messages: [
          { role: 'system', content: activeChat.value.systemPrompt || defaultSystemPrompt },
          ...activeChat.value.messages
            .filter((message) => message.content.trim())
            .map(({ role, content }) => ({ role, content })),
        ],
      }),
    })

    const data = await response.json()
    if (!response.ok) {
      throw new Error(data?.error?.message || 'Erro ao consultar o LLM.')
    }

    assistantMessage.content = data?.choices?.[0]?.message?.content || 'Sem resposta retornada.'
    activeChat.value.updatedAt = now()
  } catch (requestError) {
    assistantMessage.content = 'Nao consegui concluir a consulta.'
    error.value = requestError.message
  } finally {
    isSending.value = false
    await scrollToBottom()
  }
}

function handlePromptKeydown(event) {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    sendMessage()
  }
}

function openSystemPromptEditor() {
  if (!activeChat.value) return
  systemPromptDraft.value = activeChat.value.systemPrompt || defaultSystemPrompt
  promptEditorOpen.value = true
}

function saveSystemPrompt() {
  if (!activeChat.value) return
  activeChat.value.systemPrompt = systemPromptDraft.value.trim() || defaultSystemPrompt
  activeChat.value.updatedAt = now()
  promptEditorOpen.value = false
}

async function scrollToBottom() {
  await nextTick()
  if (!messagesList.value) return
  messagesList.value.scrollTo({
    top: messagesList.value.scrollHeight,
    behavior: 'smooth',
  })
}

function formatDate(value) {
  return new Intl.DateTimeFormat('pt-BR', {
    day: '2-digit',
    month: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(value))
}

function renderMarkdown(content) {
  return markdown.render(content || '')
}

watch(chats, persistChats, { deep: true })
watch(settings, persistSettings, { deep: true })
watch(activeChatId, scrollToBottom)

onMounted(() => {
  loadState()
  scrollToBottom()
})
</script>

<template>
  <main class="app-shell">
    <aside class="sidebar" aria-label="Chats salvos">
      <div class="brand">
        <span class="brand-mark">
          <Bot :size="22" />
        </span>
        <div>
          <strong>LLM Vue</strong>
          <span>{{ provider }} local cache</span>
        </div>
      </div>

      <button class="primary-action" type="button" @click="createChat">
        <MessageSquarePlus :size="18" />
        Novo chat
      </button>

      <div class="chat-list">
        <div
          v-for="chat in sortedChats"
          :key="chat.id"
          class="chat-item"
          :class="{ active: chat.id === activeChatId }"
        >
          <button class="chat-select" type="button" @click="selectChat(chat.id)">
            <span>{{ chat.title }}</span>
            <small>{{ chat.messages.length }} msg · {{ formatDate(chat.updatedAt) }}</small>
          </button>
          <button
            class="chat-delete"
            type="button"
            title="Apagar conversa"
            @click="deleteChat(chat.id)"
          >
            <Trash2 :size="16" />
          </button>
        </div>
      </div>

      <div class="sidebar-actions">
        <button type="button" title="Importar chats" @click="fileInput?.click()">
          <FileUp :size="18" />
        </button>
        <button type="button" title="Exportar todos" @click="exportAllChats">
          <Download :size="18" />
        </button>
        <input ref="fileInput" type="file" accept="application/json" hidden @change="importChats" />
      </div>
    </aside>

    <section class="workspace">
      <header class="topbar">
        <div>
          <p>Chat atual</p>
          <h1>{{ activeChat?.title || 'Novo chat' }}</h1>
        </div>

        <div class="topbar-actions">
          <button type="button" title="Editar prompt do sistema" @click="openSystemPromptEditor">
            <Settings :size="18" />
          </button>
          <button type="button" title="Renomear" @click="renameActiveChat">
            <Pencil :size="18" />
          </button>
          <button type="button" title="Exportar chat" @click="exportActiveChat">
            <Download :size="18" />
          </button>
          <button type="button" title="Limpar chat" @click="clearActiveChat">
            <Eraser :size="18" />
          </button>
        </div>
      </header>

      <section ref="messagesList" class="messages" aria-live="polite">
        <div v-if="activeChat?.messages.length === 0" class="empty-state">
          <Bot :size="42" />
          <h2>Comece uma conversa</h2>
          <p>Os chats ficam salvos somente no cache local deste navegador.</p>
        </div>

        <article
          v-for="message in activeChat?.messages"
          :key="message.id"
          class="message"
          :class="message.role"
        >
          <span class="avatar">
            <User v-if="message.role === 'user'" :size="18" />
            <Bot v-else :size="18" />
          </span>
          <div class="bubble">
            <strong>{{ message.role === 'user' ? 'Voce' : 'Assistente' }}</strong>
            <div
              v-if="message.role === 'assistant' && message.content"
              class="markdown-body"
              v-html="renderMarkdown(message.content)"
            />
            <p v-else>{{ message.content || 'Pensando...' }}</p>
          </div>
        </article>
      </section>

      <p v-if="error" class="error-message">{{ error }}</p>

      <form class="composer" @submit.prevent="sendMessage">
        <textarea
          v-model="prompt"
          rows="3"
          placeholder="Digite sua mensagem..."
          @keydown="handlePromptKeydown"
        />
        <button type="submit" :disabled="!canSend" title="Enviar">
          <Send :size="20" />
        </button>
      </form>
    </section>

    <div v-if="promptEditorOpen" class="modal-backdrop" role="presentation">
      <section class="prompt-dialog" role="dialog" aria-modal="true" aria-label="Prompt do sistema">
        <div>
          <h2>Prompt do sistema</h2>
          <p>Defina como o assistente deve responder neste chat.</p>
        </div>

        <label>
          Instrucoes
          <textarea v-model="systemPromptDraft" rows="7" autofocus />
        </label>

        <div class="dialog-actions">
          <button class="secondary-button" type="button" @click="promptEditorOpen = false">
            Cancelar
          </button>
          <button class="primary-button" type="button" @click="saveSystemPrompt">Salvar</button>
        </div>
      </section>
    </div>
  </main>
</template>
