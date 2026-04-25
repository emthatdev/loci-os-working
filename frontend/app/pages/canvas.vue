<template>
  <div class="canvas-page">
    <ClientOnly>
      <TresCanvas window-size clear-color="#05030f" :antialias="true">
        <TresPerspectiveCamera :position="[0, 0, 22]" :fov="55" />
        <OrbitControls
          :enable-damping="true"
          :damping-factor="0.06"
          :enable-pan="true"
          :enable-zoom="true"
          :min-distance="3"
          :max-distance="80"
        />

        <TresAmbientLight color="#ffffff" :intensity="2" />
        <TresPointLight :position="[15, 15, 15]" color="#a78bfa" :intensity="120" />
        <TresPointLight :position="[-15, -10, -10]" color="#6366f1" :intensity="60" />

        <TresGroup
          v-for="card in memoryCards"
          :key="card.id"
          :position="card.position"
          :rotation="card.rotation"
          :scale="hoveredId === card.id ? [1.06, 1.06, 1] : [1, 1, 1]"
        >
          <TresMesh
            @click="selectMemory(card)"
            @pointerover="hoveredId = card.id"
            @pointerout="hoveredId = null"
          >
            <TresPlaneGeometry :args="[3.2, 2]" />
            <TresMeshBasicMaterial
              :map="card.texture"
              :transparent="true"
              :opacity="hoveredId === card.id ? 1 : 0.88"
            />
          </TresMesh>
          <!-- Glow border behind the card -->
          <TresMesh :position="[0, 0, -0.01]">
            <TresPlaneGeometry :args="[3.32, 2.08]" />
            <TresMeshBasicMaterial
              :color="card.color"
              :transparent="true"
              :opacity="hoveredId === card.id ? 0.45 : 0.15"
            />
          </TresMesh>
        </TresGroup>
      </TresCanvas>
    </ClientOnly>

    <!-- Top bar -->
    <header class="topbar">
      <span class="logo">Loci OS</span>
      <button class="btn-ghost" @click="logout">Sign out</button>
    </header>

    <!-- Search bar -->
    <div class="search-wrap">
      <form class="search-bar" @submit.prevent="runSearch">
        <input
          v-model="searchQuery"
          placeholder="Search your memories…"
          :disabled="searching"
        />
        <button type="submit" :disabled="searching || !searchQuery.trim()">
          {{ searching ? '…' : '↑' }}
        </button>
      </form>
      <div v-if="searchResults.length" class="search-results">
        <div
          v-for="r in searchResults"
          :key="r.id"
          class="result-item"
          @click="focusResult(r)"
        >
          <span class="result-type">{{ r.type }}</span>
          <span class="result-content">{{ truncate(r.content ?? r.file_path ?? '', 80) }}</span>
          <span class="result-sim">{{ Math.round((r.similarity ?? 0) * 100) }}%</span>
        </div>
      </div>
    </div>

    <!-- Bottom-right actions -->
    <div class="actions">
      <button class="fab" title="Ask Loci OS" @click="chatOpen = true">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z" />
        </svg>
      </button>
      <button class="fab fab-primary" title="Add memory" @click="uploadOpen = true">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
          <line x1="12" y1="5" x2="12" y2="19" /><line x1="5" y1="12" x2="19" y2="12" />
        </svg>
      </button>
    </div>

    <!-- Selected memory detail -->
    <Transition name="pop">
      <div v-if="selected" class="detail-card" @click.self="selected = null">
        <div class="detail-inner">
          <button class="detail-close" @click="selected = null">✕</button>
          <span class="detail-type">{{ selected.type }}</span>
          <p class="detail-content">{{ selected.content ?? selected.file_path }}</p>
          <span class="detail-date">{{ formatDate(selected.created_at) }}</span>
        </div>
      </div>
    </Transition>

    <ChatPanel :open="chatOpen" @close="chatOpen = false" />
    <UploadModal :open="uploadOpen" @close="uploadOpen = false" @saved="onMemorySaved" />
  </div>
</template>

<script setup lang="ts">
import { OrbitControls } from '@tresjs/cientos'
import { CanvasTexture } from 'three'

definePageMeta({ middleware: 'auth' })

const { request } = useApi()
const auth = useAuthStore()
const router = useRouter()

interface Memory {
  id: number
  type: string
  content: string | null
  file_path: string | null
  created_at: string
  similarity?: number
}

interface MemoryCard {
  id: number
  position: [number, number, number]
  rotation: [number, number, number]
  color: string
  texture: CanvasTexture
  memory: Memory
}

const TYPE_COLORS: Record<string, string> = {
  text:  '#7c3aed',
  image: '#0ea5e9',
  pdf:   '#10b981',
}

const memoryCards = ref<MemoryCard[]>([])
const chatOpen = ref(false)
const uploadOpen = ref(false)
const hoveredId = ref<number | null>(null)
const selected = ref<Memory | null>(null)
const searchQuery = ref('')
const searchResults = ref<Memory[]>([])
const searching = ref(false)

// --- deterministic positions from ID ---
function rand(seed: number, salt: number): number {
  const x = Math.sin(seed * 127.1 + salt * 311.7) * 43758.5453
  return x - Math.floor(x)
}
function cardPosition(id: number): [number, number, number] {
  return [
    (rand(id, 0) - 0.5) * 44,
    (rand(id, 1) - 0.5) * 26,
    (rand(id, 2) - 0.5) * 34,
  ]
}
function cardRotation(id: number): [number, number, number] {
  return [
    (rand(id, 3) - 0.5) * 0.3,
    (rand(id, 4) - 0.5) * 0.5,
    (rand(id, 5) - 0.5) * 0.15,
  ]
}

// --- canvas texture for each card ---
function wrapText(
  ctx: CanvasRenderingContext2D,
  text: string,
  x: number,
  y: number,
  maxWidth: number,
  lineHeight: number,
  maxLines: number,
): void {
  const words = text.split(' ')
  let line = ''
  let drawn = 0
  for (let i = 0; i < words.length; i++) {
    const test = line ? `${line} ${words[i]}` : words[i]
    if (ctx.measureText(test).width > maxWidth && line) {
      ctx.fillText(line, x, y)
      y += lineHeight
      drawn++
      if (drawn >= maxLines - 1) {
        const rest = words.slice(i).join(' ')
        ctx.fillText(rest.length > 55 ? rest.slice(0, 55) + '…' : rest, x, y)
        return
      }
      line = words[i]
    } else {
      line = test
    }
  }
  if (line) ctx.fillText(line, x, y)
}

function makeTexture(memory: Memory): CanvasTexture {
  const color = TYPE_COLORS[memory.type] ?? '#7c3aed'
  const W = 512
  const H = 320
  const cvs = document.createElement('canvas')
  cvs.width = W
  cvs.height = H
  const ctx = cvs.getContext('2d')!

  // Background
  ctx.fillStyle = '#130e25'
  ctx.beginPath()
  ctx.roundRect(0, 0, W, H, 14)
  ctx.fill()

  // Top accent stripe
  ctx.fillStyle = color
  ctx.fillRect(0, 0, W, 6)

  // Subtle border
  ctx.strokeStyle = color + '60'
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.roundRect(0.75, 0.75, W - 1.5, H - 1.5, 14)
  ctx.stroke()

  // Type badge
  ctx.fillStyle = color + '28'
  ctx.beginPath()
  ctx.roundRect(18, 18, 72, 24, 5)
  ctx.fill()
  ctx.fillStyle = color
  ctx.font = 'bold 12px system-ui, sans-serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(memory.type.toUpperCase(), 54, 30)

  // Content text
  ctx.fillStyle = '#d4c8f0'
  ctx.font = '15px system-ui, sans-serif'
  ctx.textAlign = 'left'
  ctx.textBaseline = 'top'
  const text = memory.content ?? memory.file_path ?? '(no content)'
  wrapText(ctx, text, 18, 60, W - 36, 26, 8)

  // Date (bottom-right)
  const date = new Date(memory.created_at).toLocaleDateString(undefined, { dateStyle: 'short' })
  ctx.fillStyle = color + 'aa'
  ctx.font = '11px system-ui, sans-serif'
  ctx.textAlign = 'right'
  ctx.textBaseline = 'bottom'
  ctx.fillText(date, W - 18, H - 14)

  return new CanvasTexture(cvs)
}

function buildCard(memory: Memory): MemoryCard {
  return {
    id: memory.id,
    position: cardPosition(memory.id),
    rotation: cardRotation(memory.id),
    color: TYPE_COLORS[memory.type] ?? '#7c3aed',
    texture: makeTexture(memory),
    memory,
  }
}

// Fetch on mount (client only — $api plugin is client-only)
onMounted(async () => {
  const res = await request<Memory[]>('/memories')
  if (res.success) {
    memoryCards.value = (res.data ?? []).map(buildCard)
  }
})

onUnmounted(() => {
  memoryCards.value.forEach(c => c.texture.dispose())
})

function selectMemory(card: MemoryCard) {
  selected.value = card.memory
}

async function runSearch() {
  const q = searchQuery.value.trim()
  if (!q) return
  searching.value = true
  const res = await request<Memory[]>('/memories/search', {
    method: 'POST',
    body: { query: q },
  })
  searching.value = false
  searchResults.value = res.success ? (res.data ?? []) : []
}

function focusResult(r: Memory) {
  selected.value = r
  searchResults.value = []
  searchQuery.value = ''
}

function onMemorySaved(memory: Memory) {
  memoryCards.value.unshift(buildCard(memory))
}

function truncate(s: string, n: number) {
  return s.length > n ? s.slice(0, n) + '…' : s
}

function formatDate(iso: string) {
  return new Date(iso).toLocaleDateString(undefined, { dateStyle: 'medium' })
}

async function logout() {
  await request('/logout', { method: 'POST' })
  auth.logout()
  router.push('/')
}
</script>

<style scoped>
.canvas-page {
  position: fixed; inset: 0;
  background: var(--bg);
  overflow: hidden;
}

/* Top bar */
.topbar {
  position: fixed; top: 0; left: 0; right: 0;
  display: flex; align-items: center; justify-content: space-between;
  padding: 1rem 1.5rem;
  z-index: 10;
  background: linear-gradient(to bottom, rgba(5,3,15,0.8) 0%, transparent 100%);
}
.logo {
  font-size: 1.125rem; font-weight: 700; letter-spacing: -0.02em;
  background: linear-gradient(135deg, #a78bfa, #6366f1);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent;
}
.btn-ghost {
  background: none; border: 1px solid var(--border); color: var(--muted);
  border-radius: 8px; padding: 0.4rem 1rem; font-size: 0.8125rem;
  cursor: pointer; transition: all 0.2s;
}
.btn-ghost:hover { color: var(--text); border-color: var(--text); }

/* Search */
.search-wrap {
  position: fixed; bottom: 2rem; left: 50%;
  transform: translateX(-50%);
  width: min(480px, calc(100vw - 8rem));
  z-index: 10;
}
.search-bar {
  display: flex; gap: 0.5rem;
  background: var(--glass);
  border: 1px solid var(--border);
  border-radius: 999px;
  padding: 0.5rem 0.5rem 0.5rem 1.25rem;
  backdrop-filter: blur(16px);
  box-shadow: 0 8px 32px rgba(0,0,0,0.4);
}
.search-bar input {
  flex: 1; background: none; border: none; color: var(--text);
  font-size: 0.9375rem; outline: none;
}
.search-bar input::placeholder { color: var(--muted); opacity: 0.6; }
.search-bar button {
  width: 36px; height: 36px; border-radius: 50%;
  background: var(--accent); color: #fff; border: none;
  font-size: 1.1rem; cursor: pointer; flex-shrink: 0;
  transition: background 0.2s;
}
.search-bar button:hover:not(:disabled) { background: #6d28d9; }
.search-bar button:disabled { opacity: 0.4; cursor: not-allowed; }

.search-results {
  background: var(--glass);
  border: 1px solid var(--border);
  border-radius: 12px;
  margin-top: 0.5rem;
  overflow: hidden;
  backdrop-filter: blur(16px);
  box-shadow: 0 8px 32px rgba(0,0,0,0.4);
}
.result-item {
  display: flex; align-items: center; gap: 0.75rem;
  padding: 0.75rem 1rem;
  cursor: pointer; transition: background 0.15s;
  border-bottom: 1px solid var(--border);
}
.result-item:last-child { border-bottom: none; }
.result-item:hover { background: rgba(255,255,255,0.05); }
.result-type {
  font-size: 0.6875rem; font-weight: 600; text-transform: uppercase;
  letter-spacing: 0.05em; color: var(--accent); flex-shrink: 0;
}
.result-content { flex: 1; font-size: 0.875rem; color: var(--text); overflow: hidden; white-space: nowrap; text-overflow: ellipsis; }
.result-sim { font-size: 0.75rem; color: var(--muted); flex-shrink: 0; }

/* FABs */
.actions {
  position: fixed; bottom: 2rem; right: 1.5rem;
  display: flex; flex-direction: column; gap: 0.75rem;
  z-index: 10;
}
.fab {
  width: 44px; height: 44px; border-radius: 50%;
  background: var(--glass); border: 1px solid var(--border);
  color: var(--text); cursor: pointer;
  display: flex; align-items: center; justify-content: center;
  backdrop-filter: blur(12px);
  transition: all 0.2s;
  box-shadow: 0 4px 16px rgba(0,0,0,0.3);
}
.fab:hover { border-color: var(--accent); color: var(--accent); }
.fab-primary { background: var(--accent); border-color: transparent; color: #fff; }
.fab-primary:hover { background: #6d28d9; color: #fff; }

/* Detail card */
.detail-card {
  position: fixed; inset: 0;
  display: flex; align-items: center; justify-content: center;
  z-index: 50; padding: 1rem;
}
.detail-inner {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 1.25rem;
  padding: 1.75rem;
  width: 100%; max-width: 440px;
  position: relative;
  box-shadow: 0 24px 64px rgba(0,0,0,0.6);
}
.detail-close {
  position: absolute; top: 1rem; right: 1rem;
  background: none; border: none; color: var(--muted);
  font-size: 1rem; cursor: pointer; transition: color 0.2s;
}
.detail-close:hover { color: var(--text); }
.detail-type {
  display: inline-block; font-size: 0.6875rem; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.06em;
  color: var(--accent); margin-bottom: 0.75rem;
}
.detail-content {
  font-size: 0.9375rem; line-height: 1.6; color: var(--text);
  white-space: pre-wrap; word-break: break-word; margin-bottom: 1rem;
}
.detail-date { font-size: 0.8125rem; color: var(--muted); }

.pop-enter-active, .pop-leave-active { transition: opacity 0.2s, transform 0.2s; }
.pop-enter-from, .pop-leave-to { opacity: 0; transform: scale(0.96); }
</style>
