<script setup lang="ts">
import { computed, ref } from 'vue'

type Profile = { id: number; name: string; controller: string; panelUrl: string }
const profiles = ref<Profile[]>([])
const activeId = ref<number | null>(null)
const showAdd = ref(false)
const name = ref('')
const controller = ref('')
const panelUrl = ref('')
const activeTab = ref('overview')
function navigate(tab: string, target: string) {
  activeTab.value = tab
  if (target === 'add') { showAdd.value = true; return }
  document.getElementById(target)?.scrollIntoView({ behavior: 'smooth', block: 'start' })
}
const current = computed(() => profiles.value.find(p => p.id === activeId.value) ?? null)
function normalizeUrl(value: string) {
  const v = value.trim()
  if (!v) return ''
  try { const u = new URL(v.includes('://') ? v : 'http://' + v); if (!['http:', 'https:'].includes(u.protocol)) return ''; return u.toString().replace(/\/$/, '') } catch { return '' }
}
function addProfile() {
  const c = normalizeUrl(controller.value)
  const p = normalizeUrl(panelUrl.value)
  if (!c || !p) return
  const item: Profile = { id: Date.now(), name: name.value.trim() || c, controller: c, panelUrl: p }
  profiles.value.push(item); activeId.value = item.id
  name.value = ''; controller.value = ''; panelUrl.value = ''; showAdd.value = false
}
function removeProfile(id: number) {
  profiles.value = profiles.value.filter(p => p.id !== id)
  if (activeId.value === id) activeId.value = profiles.value[0]?.id ?? null
}
</script>

<template>
  <main class="shell">
    <header class="topbar">
      <div class="brand"><div class="logo">M</div><div><strong>Mihomo</strong><span>MultiPanel</span></div></div>
      <div class="status"><i></i> CONTROLLERS</div>
    </header>
    <section id="overview" class="hero">
      <div class="eyebrow">YOUR CONNECTION HUB</div>
      <h1>One place for<br/><em>every dashboard.</em></h1>
      <p>Save controller addresses and use your preferred dashboard URL.</p>
    </section>
    <section id="connections" class="connection card">
      <div class="section-heading"><div><span class="eyebrow">CONTROLLERS</span><h2>Your connections</h2></div><span class="pill">{{ profiles.length }} saved</span></div>
      <div v-if="profiles.length" class="panel-list">
        <button v-for="profile in profiles" :key="profile.id" class="panel-card" :class="{selected: activeId === profile.id}" @click="activeId = profile.id">
          <div class="panel-mark">M</div><div class="panel-copy"><strong>{{ profile.name }}</strong><span>{{ profile.controller }}</span><span class="hint">{{ profile.panelUrl }}</span></div><div class="arrow">↗</div>
        </button>
      </div>
      <div v-else class="empty">No saved connections yet. Add your Mihomo controller and dashboard URL.</div>
      <button class="add-button" @click="showAdd = true"><span class="plus">＋</span> Add connection</button>
    </section>
    <section v-if="current" id="dashboard" class="panels">
      <div class="section-heading"><div><span class="eyebrow">DASHBOARD</span><h2>{{ current.name }}</h2></div></div>
      <div class="preview card"><div class="preview-head"><span class="eyebrow">PANEL URL</span><span class="pill">Custom</span></div><h3>MetaCubeXD</h3><p>{{ current.panelUrl }}</p><p class="hint">Panel loading and API integration are not implemented yet.</p></div>
    </section>
    <footer>MIHOMO MULTIPANEL <span>•</span> EARLY PROTOTYPE</footer>
    <nav class="bottom-nav" aria-label="Main navigation">
      <button :class="{ active: activeTab === 'overview' }" @click="navigate('overview', 'overview')"><svg viewBox="0 0 24 24" aria-hidden="true"><path d="m3 10 9-7 9 7v10a1 1 0 0 1-1 1h-5v-7H9v7H4a1 1 0 0 1-1-1z"/></svg><span>Обзор</span></button>
      <button :class="{ active: activeTab === 'connections' }" @click="navigate('connections', 'connections')"><svg viewBox="0 0 24 24" aria-hidden="true"><rect x="3" y="4" width="18" height="6" rx="2"/><rect x="3" y="14" width="18" height="6" rx="2"/><path d="M7 7h.01M7 17h.01M11 7h6M11 17h6"/></svg><span>Панели</span></button>
      <button @click="navigate('add', 'add')"><svg viewBox="0 0 24 24" aria-hidden="true"><path d="M12 5v14M5 12h14"/></svg><span>Добавить</span></button>
    </nav>
    <div v-if="showAdd" class="modal-backdrop" @click.self="showAdd = false">
      <section class="modal card">
        <div class="section-heading"><div><span class="eyebrow">NEW CONNECTION</span><h2>Add controller</h2></div><button class="close" @click="showAdd = false">×</button></div>
        <label for="profile-name">Name (optional)</label><input id="profile-name" v-model="name" placeholder="Home, VPS, Router…"/>
        <label for="controller-url">Mihomo controller address</label><input id="controller-url" v-model="controller" placeholder="http://192.168.1.1:9090"/>
        <label for="panel-url">Dashboard URL</label><input id="panel-url" v-model="panelUrl" placeholder="http://192.168.1.1:9090/ui/metacubexd"/>
        <p class="hint">Enter the controller's external address and the URL of the panel configured on it. URLs must start with http:// or https:// (or will be treated as http://).</p>
        <div class="modal-actions"><button class="secondary" @click="showAdd = false">Cancel</button><button @click="addProfile" :disabled="!normalizeUrl(controller) || !normalizeUrl(panelUrl)">Save</button></div>
      </section>
    </div>
  </main>
</template>
