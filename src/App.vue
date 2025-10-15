<template>
  <main class="container">
    <header class="topbar">
      <h1>To-Do PWA</h1>

      <!-- Indicador online/offline -->
      <span :class="['badge', online ? 'ok' : 'warn']" aria-live="polite">
        {{ online ? 'Online' : 'Offline' }}
      </span>
    </header>

    <!-- Instalación PWA -->
    <div v-if="canInstall" class="install">
      <button class="btn install-btn" @click="instalar">Instalar app</button>
    </div>

    <!-- Formulario nueva tarea -->
    <section class="card">
      <div class="row">
        <input
          v-model="texto"
          @keyup.enter="agregar"
          class="input"
          placeholder="Nueva tarea…"
          aria-label="Nueva tarea"
        />
        <select v-model="priority" class="select" aria-label="Prioridad">
          <option value="normal">Prioridad: Normal</option>
          <option value="alta">Prioridad: Alta</option>
          <option value="baja">Prioridad: Baja</option>
        </select>
        <input v-model="due" type="date" class="input-date" aria-label="Fecha límite" />
        <button class="btn" @click="agregar">Agregar</button>
      </div>
      <p class="muted">Las tareas se guardan automáticamente y funcionan sin internet.</p>
    </section>

    <!-- Controles de listado -->
    <section class="controls">
      <div class="row wrap">
        <div class="tabs" role="tablist" aria-label="Filtros">
          <button :class="['tab', filter==='all' && 'active']" @click="setFilter('all')" role="tab">Todas</button>
          <button :class="['tab', filter==='active' && 'active']" @click="setFilter('active')" role="tab">Pendientes</button>
          <button :class="['tab', filter==='completed' && 'active']" @click="setFilter('completed')" role="tab">Completadas</button>
        </div>

        <input v-model="search" class="input search" placeholder="Buscar…" aria-label="Buscar tareas" />

        <select v-model="sortBy" class="select" aria-label="Orden">
          <option value="newest">Más nuevas</option>
          <option value="oldest">Más antiguas</option>
          <option value="dueAsc">Fecha límite (↑)</option>
          <option value="dueDesc">Fecha límite (↓)</option>
          <option value="prio">Prioridad</option>
        </select>
      </div>

      <div class="row wrap">
        <button class="btn ghost" @click="toggleAll(true)">Marcar todas</button>
        <button class="btn ghost" @click="toggleAll(false)">Desmarcar todas</button>
        <button class="btn danger ghost" :disabled="!hasCompleted" @click="confirmClearCompleted">
          Borrar completadas
        </button>
      </div>
    </section>

    <!-- Lista -->
    <ul class="list" v-if="visibleTareas.length">
      <li v-for="t in visibleTareas" :key="t.id" class="item">
        <label class="left">
          <input type="checkbox" :checked="t.completed" @change="toggle(t.id)" aria-label="Completar tarea" />
          <div class="textblock">
            <template v-if="editId !== t.id">
              <span :class="['texto', t.completed && 'done']" @dblclick="startEdit(t)">{{ t.texto }}</span>
              <div class="meta">
                <span :class="['pill', prioClass(t.priority)]">{{ t.priority }}</span>
                <span v-if="t.due" class="pill due" :class="{ overdue: isOverdue(t) }">
                  vence {{ formatDate(t.due) }}
                </span>
              </div>
            </template>
            <template v-else>
              <input v-model="editText" class="input small" @keyup.enter="saveEdit(t)" @keyup.esc="cancelEdit" />
            </template>
          </div>
        </label>

        <div class="actions">
          <button class="btn ghost" v-if="editId !== t.id" @click="startEdit(t)">Editar</button>
          <button class="btn" v-else @click="saveEdit(t)">Guardar</button>
          <button class="btn danger" @click="confirmDelete(t.id)">Eliminar</button>
        </div>
      </li>
    </ul>

    <div v-else class="empty">
      <p>No hay tareas que coincidan con el filtro/búsqueda.</p>
    </div>

    <p class="hint">Modo offline listo cuando instales la app.</p>

    <!-- Modal de confirmación -->
    <div v-if="confirm.show" class="modal-backdrop" role="dialog" aria-modal="true">
      <div class="modal">
        <h3>Confirmar</h3>
        <p>{{ confirm.message }}</p>
        <div class="row end">
          <button class="btn ghost" @click="confirmCancel">Cancelar</button>
          <button class="btn danger" @click="confirmOk">Aceptar</button>
        </div>
      </div>
    </div>
  </main>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

/* ---------- Estado ---------- */
const texto = ref('')
const priority = ref('normal') // normal | alta | baja
const due = ref('')

const tareas = ref(load('tareas', []))
const filter = ref(load('filter', 'all')) // all | active | completed
const search = ref(load('search', ''))
const sortBy = ref(load('sortBy', 'newest'))

// edición inline
const editId = ref(null)
const editText = ref('')

// instalación PWA
const deferredPrompt = ref(null)
const canInstall = ref(false)

// conectividad
const online = ref(navigator.onLine)

// confirm modal
const confirm = ref({ show: false, message: '', onOk: null })

/* ---------- Persistencia ---------- */
watch([tareas, filter, search, sortBy], () => {
  save('tareas', tareas.value)
  save('filter', filter.value)
  save('search', search.value)
  save('sortBy', sortBy.value)
}, { deep: true })

function load(key, fallback) {
  try { return JSON.parse(localStorage.getItem(key)) ?? fallback } catch { return fallback }
}
function save(key, val) {
  localStorage.setItem(key, JSON.stringify(val))
}

/* ---------- Lógica de tareas ---------- */
function agregar() {
  const t = texto.value.trim()
  if (!t) return
  tareas.value.push({
    id: Date.now(),
    texto: t,
    completed: false,
    priority: priority.value,
    due: due.value || null,
    createdAt: Date.now(),
    updatedAt: Date.now()
  })
  texto.value = ''
  due.value = ''
  priority.value = 'normal'
}

function toggle(id) {
  const i = tareas.value.findIndex(x => x.id === id)
  if (i >= 0) {
    tareas.value[i].completed = !tareas.value[i].completed
    tareas.value[i].updatedAt = Date.now()
  }
}

function startEdit(t) {
  editId.value = t.id
  editText.value = t.texto
}

function saveEdit(t) {
  const val = editText.value.trim()
  if (!val) return cancelEdit()
  t.texto = val
  t.updatedAt = Date.now()
  editId.value = null
  editText.value = ''
}

function cancelEdit() {
  editId.value = null
  editText.value = ''
}

function eliminar(id) {
  tareas.value = tareas.value.filter(t => t.id !== id)
}

function toggleAll(state) {
  tareas.value = tareas.value.map(t => ({ ...t, completed: !!state }))
}

const hasCompleted = computed(() => tareas.value.some(t => t.completed))
function clearCompleted() {
  tareas.value = tareas.value.filter(t => !t.completed)
}

/* ---------- Confirm dialogs ---------- */
function confirmDelete(id) {
  confirm.value = {
    show: true,
    message: '¿Eliminar esta tarea? Esta acción no se puede deshacer.',
    onOk: () => eliminar(id)
  }
}
function confirmClearCompleted() {
  confirm.value = {
    show: true,
    message: '¿Eliminar todas las tareas completadas?',
    onOk: clearCompleted
  }
}
function confirmOk() {
  confirm.value.onOk?.()
  confirm.value = { show: false, message: '', onOk: null }
}
function confirmCancel() {
  confirm.value = { show: false, message: '', onOk: null }
}

/* ---------- Filtros/orden ---------- */
function setFilter(f) { filter.value = f }

const visibleTareas = computed(() => {
  let list = [...tareas.value]

  // filtro estado
  if (filter.value === 'active') list = list.filter(t => !t.completed)
  if (filter.value === 'completed') list = list.filter(t => t.completed)

  // búsqueda
  const q = search.value.trim().toLowerCase()
  if (q) list = list.filter(t => t.texto.toLowerCase().includes(q))

  // orden
  switch (sortBy.value) {
    case 'oldest': list.sort((a,b) => a.createdAt - b.createdAt); break
    case 'dueAsc': list.sort((a,b) => (a.due||'') > (b.due||'') ? 1 : -1); break
    case 'dueDesc': list.sort((a,b) => (a.due||'') < (b.due||'') ? 1 : -1); break
    case 'prio': list.sort((a,b) => prioWeight(b.priority) - prioWeight(a.priority)); break
    default: list.sort((a,b) => b.createdAt - a.createdAt) // newest
  }
  return list
})

function prioWeight(p) {
  return p === 'alta' ? 2 : p === 'normal' ? 1 : 0
}
function prioClass(p) {
  return p === 'alta' ? 'pill-high' : p === 'baja' ? 'pill-low' : 'pill-normal'
}
function isOverdue(t) {
  return !!t.due && !t.completed && new Date(t.due) < new Date(new Date().toDateString())
}
function formatDate(d) {
  try { return new Date(d).toLocaleDateString() } catch { return d }
}

/* ---------- Instalación PWA ---------- */
onMounted(() => {
  const onBip = (e) => {
    e.preventDefault()
    deferredPrompt.value = e
    canInstall.value = true
  }
  window.addEventListener('beforeinstallprompt', onBip)
  window.addEventListener('appinstalled', () => {
    canInstall.value = false
    deferredPrompt.value = null
  })

  const setOnline = () => (online.value = navigator.onLine)
  window.addEventListener('online', setOnline)
  window.addEventListener('offline', setOnline)

  cleanup.push(() => {
    window.removeEventListener('beforeinstallprompt', onBip)
    window.removeEventListener('online', setOnline)
    window.removeEventListener('offline', setOnline)
  })
})

function instalar() {
  const evt = deferredPrompt.value
  if (!evt) return
  evt.prompt()
  evt.userChoice?.then(() => {
    deferredPrompt.value = null
    canInstall.value = false
  })
}

/* ---------- Cleanup ---------- */
const cleanup = []
onBeforeUnmount(() => cleanup.forEach(fn => fn()))
</script>

<style>
:root { color-scheme: dark; }
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, ui-sans-serif, Segoe UI, Roboto, Arial; background:#0b0b12; color:#e7e7ea; }

.container { max-width: 780px; margin: 32px auto 80px; padding: 0 16px; }

.topbar { display:flex; align-items:center; justify-content:space-between; gap:12px; margin-bottom:12px; }
h1 { margin: 0; font-size: 28px; letter-spacing:.2px; }

.badge { padding:6px 10px; border-radius:999px; font-size:12px; border:1px solid #2a2a33; background:#13131b; }
.badge.ok { border-color:#1e3a2a; background:#112016; color:#86efac; }
.badge.warn { border-color:#3a1e1e; background:#201111; color:#fca5a5; }

.install { margin: 8px 0 16px; }
.install-btn { background:#22c55e; color:#07130b; }

.card { border:1px solid #23232c; background:#12121a; border-radius:14px; padding:16px; }
.muted { opacity:.7; margin:8px 0 0; font-size:13px; }

.controls { margin: 16px 0; display:grid; gap:10px; }
.wrap { flex-wrap: wrap; }
.row { display:flex; gap:8px; align-items:center; }
.row.end { justify-content:flex-end; }

.input { flex: 1; padding: 12px; border-radius: 10px; border: 1px solid #2a2a33; background: #0f0f16; color: inherit; }
.input.small { padding:10px; }
.input-date { padding: 12px; border-radius: 10px; border: 1px solid #2a2a33; background: #0f0f16; color: inherit; }
.select { padding: 12px; border-radius: 10px; border: 1px solid #2a2a33; background: #0f0f16; color: inherit; }

.btn { padding: 12px 14px; border: 0; border-radius: 10px; background: #0ea5e9; color: #06111a; font-weight: 700; cursor: pointer; }
.btn:hover { filter: brightness(1.05); }
.btn.ghost { background: #15151f; color:#e7e7ea; border:1px solid #2a2a33; }
.btn.ghost:hover { background:#181824; }
.btn.danger { background: #ef4444; color: #fff; }

.tabs { display:flex; gap:6px; background:#101018; padding:6px; border-radius:12px; border:1px solid #23232c; }
.tab { background:transparent; color:#c9c9d1; border:0; padding:8px 12px; border-radius:10px; cursor:pointer; }
.tab.active { background:#0ea5e9; color:#06111a; font-weight:700; }

.search { min-width: 180px; }

.list { list-style: none; padding: 0; margin: 12px 0 0; display: grid; gap: 10px; }
.item { display:flex; justify-content:space-between; align-items:center; gap: 12px; padding: 12px; border: 1px solid #23232c; border-radius: 12px; background: #13131b; }
.left { display:flex; gap:10px; align-items:flex-start; }
.textblock { display:flex; flex-direction:column; gap:6px; }
.texto { font-size:16px; }
.texto.done { text-decoration: line-through; opacity:.6; }

.meta { display:flex; gap:8px; flex-wrap:wrap; }
.pill { font-size:12px; padding:4px 8px; border-radius:999px; border:1px solid #2a2a33; background:#0f0f16; }
.pill-high { border-color:#3a1e1e; background:#201111; color:#fca5a5; }
.pill-normal { border-color:#1e2f3a; background:#0f171f; color:#7dd3fc; }
.pill-low { border-color:#1e3a2a; background:#112016; color:#86efac; }
.pill.due { border-color:#35323f; background:#1a1824; color:#c3b8ff; }
.pill.due.overdue { border-color:#3a1e1e; background:#201111; color:#fca5a5; }

.actions { display:flex; gap:8px; }

.empty { border:1px dashed #2a2a33; background:#101018; border-radius:12px; padding:24px; text-align:center; opacity:.9; }

.hint { opacity: .7; margin-top: 16px; font-size: 13px; }

/* Modal */
.modal-backdrop { position:fixed; inset:0; background:rgba(0,0,0,.5); display:grid; place-items:center; padding:16px; }
.modal { width:min(520px, 100%); background:#12121a; border:1px solid #2a2a33; border-radius:14px; padding:18px; }
.modal h3 { margin:0 0 8px; }
.modal p { margin:0 0 14px; opacity:.9; }
</style>
