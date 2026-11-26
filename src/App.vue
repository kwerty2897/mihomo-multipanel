<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type Profile = { id: number; name: string; controller: string; panelUrl: string }
const STORAGE_KEY = 'mihomo-multipanel-profiles-v1'
const SETTINGS_KEY = 'mihomo-multipanel-settings-v1'
type Theme = 'system' | 'light' | 'dark'
const theme = ref<Theme>((() => {
  try { const value = JSON.parse(localStorage.getItem(SETTINGS_KEY) || '{}').theme; return ['system', 'light', 'dark'].includes(value) ? value : 'system' } catch { return 'system' }
})())
const panelOpen = ref(false)
const panelLoading = ref(false)
const showSettings = ref(false)
const showAbout = ref(false)
const importInput = ref<HTMLInputElement | null>(null)
const importMessage = ref('')
watch(theme, value => {
  try { localStorage.setItem(SETTINGS_KEY, JSON.stringify({ theme: value })) } catch { /* storage may be unavailable */ }
  if (value === 'system') document.documentElement.removeAttribute('data-theme')
  else document.documentElement.setAttribute('data-theme', value)
  const systemBars = (window as Window & { AndroidSystemBars?: { setTheme: (theme: string) => void } }).AndroidSystemBars
  systemBars?.setTheme(value)
}, { immediate: true })
function loadSavedProfiles(): Profile[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return []
    const parsed: unknown = JSON.parse(raw)
    return Array.isArray(parsed) ? parsed.filter((item): item is Profile =>
      item && typeof item.id === 'number' && typeof item.name === 'string' &&
      typeof item.controller === 'string' && typeof item.panelUrl === 'string'
    ) : []
  } catch { return [] }
}
const profiles = ref<Profile[]>(loadSavedProfiles())
const activeId = ref<number | null>(profiles.value[0]?.id ?? null)
watch(profiles, value => {
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(value)) } catch { /* storage may be unavailable */ }
}, { deep: true })
const showAdd = ref(false)
const name = ref('')
const controller = ref('')
const panelUrl = ref('')
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
function exportSettings() {
  const backup = { app: 'mihomo-multipanel', version: 1, exportedAt: new Date().toISOString(), settings: { theme: theme.value }, profiles: profiles.value }
  const blob = new Blob([JSON.stringify(backup, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a'); link.href = url; link.download = 'mihomo-multipanel-backup-' + new Date().toISOString().slice(0, 10) + '.json'
  document.body.appendChild(link); link.click(); link.remove(); URL.revokeObjectURL(url)
  importMessage.value = 'Backup exported.'
}
async function importSettings(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return
  try {
    const data: unknown = JSON.parse(await file.text())
    if (!data || typeof data !== 'object') throw new Error('Invalid backup file.')
    const backup = data as { app?: unknown; version?: unknown; profiles?: unknown; settings?: { theme?: unknown } }
    if (backup.app !== 'mihomo-multipanel' || backup.version !== 1 || !Array.isArray(backup.profiles)) throw new Error('Unsupported backup format.')
    const restored = backup.profiles.filter((item): item is Profile => !!item && typeof item === 'object' && typeof (item as Profile).id === 'number' && typeof (item as Profile).name === 'string' && typeof (item as Profile).controller === 'string' && typeof (item as Profile).panelUrl === 'string')
    if (restored.length !== backup.profiles.length) throw new Error('Backup contains invalid connection entries.')
    if (!window.confirm('Import ' + restored.length + ' connection(s) and settings? This will replace current saved data.')) return
    profiles.value = restored
    activeId.value = restored[0]?.id ?? null
    const importedTheme = backup.settings?.theme
    theme.value = ['system', 'light', 'dark'].includes(String(importedTheme)) ? importedTheme as Theme : 'system'
    importMessage.value = 'Backup imported successfully.'
  } catch (error) { importMessage.value = error instanceof Error ? error.message : 'Could not import backup.' }
  finally { input.value = '' }
}
function openPanel() {
  if (!current.value) return
  panelLoading.value = true
  panelOpen.value = true
}
function closePanel() { panelOpen.value = false; panelLoading.value = false }
function reloadPanel() {
  if (!current.value) return
  panelLoading.value = true
  const frame = document.querySelector<HTMLIFrameElement>('.panel-frame')
  if (frame) frame.src = current.value.panelUrl
}
function removeProfile(id: number) {
  profiles.value = profiles.value.filter(p => p.id !== id)
  if (activeId.value === id) activeId.value = profiles.value[0]?.id ?? null
}
</script>

<template>
  <main class="shell">
    <section v-if="panelOpen && current" class="panel-viewer">
      <header class="panel-toolbar">
        <button class="panel-toolbar-button" aria-label="Back to connections" @click="closePanel">‹</button>
        <div class="panel-toolbar-title"><strong>{{ current.name }}</strong><span>{{ current.panelUrl }}</span></div>
        <button class="panel-toolbar-button" aria-label="Reload panel" @click="reloadPanel">↻</button>
      </header>
      <div v-if="panelLoading" class="panel-loading">Loading dashboard…</div>
      <iframe :key="current.id" class="panel-frame" :src="current.panelUrl" :title="current.name + ' dashboard'" allow="clipboard-read; clipboard-write; fullscreen" allowfullscreen @load="panelLoading = false"></iframe>
    </section>
    <template v-else>
    <section id="overview" class="hero">
      <div class="hero-top"><div class="eyebrow">YOUR CONNECTION HUB</div><button class="settings-button" aria-label="Settings" title="Settings" @click="showSettings = true">⚙</button></div>
      <h1>One place for<br/><em>every dashboard.</em></h1>
      <p>Save controller addresses and use your preferred dashboard URL.</p>
    </section>
    <section id="connections" class="connection card">
      <div class="section-heading"><div><span class="eyebrow">CONTROLLERS</span><h2>Your connections</h2></div><span class="pill">{{ profiles.length }} saved</span></div>
      <div v-if="profiles.length" class="panel-list">
        <div v-for="profile in profiles" :key="profile.id" class="panel-card" :class="{selected: activeId === profile.id}" role="button" tabindex="0" @click="activeId = profile.id" @keydown.enter="activeId = profile.id">
          <div class="panel-mark">M</div><div class="panel-copy"><strong>{{ profile.name }}</strong><span>{{ profile.controller }}</span><span class="hint">{{ profile.panelUrl }}</span></div><button class="remove-profile" aria-label="Remove connection" @click.stop="removeProfile(profile.id)">×</button><div class="arrow">›</div>
        </div>
      </div>
      <div v-else class="empty">No saved connections yet. Add your Mihomo controller and dashboard URL.</div>
      <button class="add-button" @click="showAdd = true"><span class="plus">＋</span> Add connection</button>
    </section>
    <section v-if="current" id="dashboard" class="panels">
      <div class="section-heading"><div><span class="eyebrow">DASHBOARD</span><h2>{{ current.name }}</h2></div></div>
      <div class="preview card"><div class="preview-head"><span class="eyebrow">PANEL URL</span><span class="pill">Ready</span></div><h3>Dashboard</h3><p>{{ current.panelUrl }}</p><button class="open-panel-button" @click="openPanel">Open dashboard <span>↗</span></button><p class="hint">The dashboard opens inside the app. Its own update mechanism remains unchanged.</p></div>
    </section>
    <footer>MIHOMO MULTIPANEL <span>•</span> EARLY PROTOTYPE</footer>
    </template>

    <div v-if="showSettings" class="modal-backdrop" @click.self="showSettings = false">
      <section class="modal card settings-modal">
        <div class="section-heading"><div><span class="eyebrow">PREFERENCES & BACKUP</span><h2>Settings</h2></div><button class="close" aria-label="Close settings" @click="showSettings = false">×</button></div>
        <label for="theme-choice">Appearance</label>
        <select id="theme-choice" v-model="theme" class="settings-select">
          <option value="system">Use system theme</option>
          <option value="light">Light</option>
          <option value="dark">Dark</option>
        </select>
        <div class="settings-divider"></div>
        <div class="backup-title">Backup & restore</div>
        <p class="hint">Export or restore all saved controller connections, panel URLs and appearance settings in one JSON file.</p>
        <button class="backup-button" @click="exportSettings">Export settings</button>
        <button class="backup-button secondary-backup" @click="importInput?.click()">Import settings</button>
        <input ref="importInput" class="file-input" type="file" accept=".json,application/json" @change="importSettings" />
        <p v-if="importMessage" class="import-message" role="status">{{ importMessage }}</p>
        <p class="hint security-note">The backup contains your saved addresses. Keep the file private if it includes internal network details.</p>
        <div class="settings-divider"></div>
        <button class="about-button" @click="showAbout = true">About</button>
      </section>
    </div>

    <div v-if="showAbout" class="modal-backdrop about-backdrop" @click.self="showAbout = false">
      <section class="modal card about-modal">
        <div class="section-heading"><div><span class="eyebrow">APPLICATION</span><h2>About</h2></div><button class="close" aria-label="Close about" @click="showAbout = false">×</button></div>
        <div class="about-brand"><div class="about-logo" aria-label="Mihomo MultiPanel app icon"><svg viewBox="0 0 512 512" role="img" aria-hidden="true"><g fill="none" stroke="currentColor" stroke-width="16" stroke-linecap="round" stroke-linejoin="round"><path d="M256 76 405 161 405 344 256 430 107 344 107 161Z"/><path d="M107 161 256 247 405 161 M256 247 256 430"/><path d="M256 103 256 221 M145 322 235 270 M277 270 367 322"/></g></svg></div><div><strong>Mihomo MultiPanel</strong><span>Version 0.1.0</span></div></div>
        <p class="hint about-description">A simple hub for managing Mihomo controllers and web dashboards.</p>
        <a class="about-link" href="https://github.com/kwerty2897/mihomo-multipanel" target="_blank" rel="noreferrer">GitHub project <span>↗</span></a>
        <p class="about-footer">Made for the Mihomo community.</p>
      </section>
    </div>

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
