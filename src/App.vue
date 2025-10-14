<template>
  <main class="container">
    <h1>To-Do PWA</h1>

    <div class="row">
      <input
        v-model="texto"
        @keyup.enter="agregar"
        placeholder="Nueva tarea…"
        class="input"
      />
      <button class="btn" @click="agregar">Agregar</button>
    </div>

    <ul class="list">
      <li v-for="t in tareas" :key="t.id" class="item">
        <span>{{ t.texto }}</span>
        <button class="btn danger" @click="eliminar(t.id)">Eliminar</button>
      </li>
    </ul>

    <p class="hint">Modo offline listo cuando instales la app ✨</p>
  </main>
</template>

<script setup>
import { ref, watch } from 'vue'

const texto = ref('')
const tareas = ref(JSON.parse(localStorage.getItem('tareas') || '[]'))

watch(
  tareas,
  (val) => localStorage.setItem('tareas', JSON.stringify(val)),
  { deep: true }
)

function agregar() {
  const t = texto.value.trim()
  if (!t) return
  tareas.value.push({ id: Date.now(), texto: t })
  texto.value = ''
}

function eliminar(id) {
  // Sí, "eliminar" es filtrar y reasignar: en estado inmutable es *the way*
  tareas.value = tareas.value.filter((t) => t.id !== id)
}
</script>

<style>
:root { color-scheme: dark; }
body { margin: 0; font-family: system-ui, ui-sans-serif, Segoe UI, Roboto, Arial; background:#0b0b12; color:#e7e7ea; }
.container { max-width: 520px; margin: 40px auto; padding: 0 16px; }
h1 { margin: 0 0 16px; font-size: 28px; }
.row { display: flex; gap: 8px; }
.input { flex: 1; padding: 12px; border-radius: 10px; border: 1px solid #2a2a33; background: #12121a; color: inherit; }
.btn { padding: 12px 14px; border: 0; border-radius: 10px; background: #0ea5e9; color: #06111a; font-weight: 700; cursor: pointer; }
.btn:hover { filter: brightness(1.05); }
.btn.danger { background: #ef4444; color: #fff; }
.list { list-style: none; padding: 0; margin: 16px 0 0; display: grid; gap: 8px; }
.item { display: flex; justify-content: space-between; align-items: center; gap: 8px; padding: 12px; border: 1px solid #23232c; border-radius: 12px; background: #13131b; }
.hint { opacity: .7; margin-top: 12px; font-size: 14px; }
</style>
