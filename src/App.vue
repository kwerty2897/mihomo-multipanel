<script setup lang="ts">
import { ref } from 'vue'

const panels = [
  { name: 'Zashboard', detail: 'Modern Mihomo dashboard', mark: 'Z' },
  { name: 'YACD', detail: 'Yet Another Clash Dashboard', mark: 'Y' },
  { name: 'MetaCubeXD', detail: 'MetaCube dashboard', mark: 'M' }
]
const endpoint = ref('')
const active = ref('Zashboard')
const saved = ref(false)
function addController() {
  if (!endpoint.value.trim()) return
  saved.value = true
}
</script>

<template>
  <main class="shell">
    <header class="topbar">
      <div class="brand"><div class="logo">M</div><div><strong>Mihomo</strong><span>MultiPanel</span></div></div>
      <div class="status"><i></i> LOCAL UI</div>
    </header>
    <section class="hero">
      <div class="eyebrow">YOUR CONNECTION HUB</div>
      <h1>One place for<br/><em>every dashboard.</em></h1>
      <p>Connect to your Mihomo instance and choose the interface that works for you.</p>
    </section>
    <section class="connection card">
      <div class="section-heading"><div><span class="eyebrow">CONTROLLER</span><h2>Connect Mihomo</h2></div><span class="pill">Not connected</span></div>
      <label for="endpoint">Controller address</label>
      <div class="input-row"><input id="endpoint" v-model="endpoint" placeholder="http://192.168.1.1:9090" /><button @click="addController">Add</button></div>
      <p class="hint">Prototype only — connection and authentication are not yet implemented.</p>
      <p v-if="saved" class="hint">Address entered. Saving controllers will be implemented next.</p>
    </section>
    <section class="panels">
      <div class="section-heading"><div><span class="eyebrow">INTERFACES</span><h2>Choose a panel</h2></div></div>
      <div class="panel-list">
        <button v-for="panel in panels" :key="panel.name" class="panel-card" :class="{selected: active === panel.name}" @click="active = panel.name">
          <div class="panel-mark">{{ panel.mark }}</div><div class="panel-copy"><strong>{{ panel.name }}</strong><span>{{ panel.detail }}</span></div><div class="arrow">↗</div>
        </button>
      </div>
      <div class="preview card"><div class="preview-head"><span class="eyebrow">SELECTED</span><span class="pill">{{ active }}</span></div><h3>{{ active }}</h3><p>Panel integration will be added in the next stage.</p></div>
    </section>
    <footer>MIHOMO MULTIPANEL <span>•</span> EARLY PROTOTYPE</footer>
  </main>
</template>
