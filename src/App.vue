<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type Profile = { id: number; name: string; controller: string; panelUrl: string }
const STORAGE_KEY = 'mihomo-multipanel-profiles-v1'
const SETTINGS_KEY = 'mihomo-multipanel-settings-v1'
type Theme = 'system' | 'light' | 'dark'
const savedSettings = (() => {
  try { return JSON.parse(localStorage.getItem(SETTINGS_KEY) || '{}') } catch { return {} }
})()
const theme = ref<Theme>(['system', 'light', 'dark'].includes(savedSettings.theme) ? savedSettings.theme : 'system')
const animationsEnabled = ref(savedSettings.animations !== false)
const panelOpen = ref(false)
const panelLoading = ref(false)
const controllerStatus = ref<'idle' | 'checking' | 'online' | 'offline'>('idle')
const controllerStatusMessage = ref('')
const controllerCheckedAt = ref('')
let statusRequest: AbortController | null = null
const showSettings = ref(false)
const showAbout = ref(false)
const themeMenuOpen = ref(false)
const themeLabels: Record<Theme, string> = { system: 'Use system theme', light: 'Light', dark: 'Dark' }
const importInput = ref<HTMLInputElement | null>(null)
const importMessage = ref('')
watch([theme, animationsEnabled], ([themeValue, animationsValue]) => {
  try { localStorage.setItem(SETTINGS_KEY, JSON.stringify({ theme: themeValue, animations: animationsValue })) } catch { /* storage may be unavailable */ }
  const root = document.documentElement
  root.setAttribute('data-theme', themeValue === 'system' ? (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light') : themeValue)
  root.setAttribute('data-animations', animationsValue ? 'on' : 'off')
  const systemBars = (window as Window & { AndroidSystemBars?: { setTheme: (theme: string) => void } }).AndroidSystemBars
  systemBars?.setTheme(themeValue)
}, { immediate: true })
function applyAndroidPalette() {
  try {
    const bridge = (window as Window & { AndroidSystemBars?: { getPalette?: () => string } }).AndroidSystemBars
    const raw = bridge?.getPalette?.()
    if (!raw) return
    const palette = JSON.parse(raw) as Record<string, string>
    for (const [key, value] of Object.entries(palette)) {
      if (/^#[0-9A-Fa-f]{6,8}$/.test(value)) document.documentElement.style.setProperty('--monet-' + key, value)
    }
    document.documentElement.setAttribute('data-monet', 'on')
  } catch { /* unavailable on older Android versions */ }
}
applyAndroidPalette()
const systemThemeQuery = window.matchMedia('(prefers-color-scheme: dark)')
systemThemeQuery.addEventListener('change', () => {
  if (theme.value === 'system') document.documentElement.setAttribute('data-theme', systemThemeQuery.matches ? 'dark' : 'light')
})
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
const connectionUrl = ref('')
function deriveController(value: string) { const normalized = normalizeUrl(value); if (!normalized) return ''; try { return new URL(normalized).origin } catch { return '' } }
const current = computed(() => profiles.value.find(p => p.id === activeId.value) ?? null)
function normalizeUrl(value: string) {
  const v = value.trim()
  if (!v) return ''
  try { const u = new URL(v.includes('://') ? v : 'http://' + v); if (!['http:', 'https:'].includes(u.protocol)) return ''; return u.toString().replace(/\/$/, '') } catch { return '' }
}
function addProfile() {
  const p = normalizeUrl(connectionUrl.value)
  const c = deriveController(connectionUrl.value)
  if (!c || !p) return
  const item: Profile = { id: Date.now(), name: name.value.trim() || c, controller: c, panelUrl: p }
  profiles.value.push(item); activeId.value = item.id
  name.value = ''; connectionUrl.value = ''; showAdd.value = false
}
function exportSettings() {
  const payload = { settings: { theme: theme.value, animations: animationsEnabled.value }, profiles: profiles.value }
  const encoded = btoa(unescape(encodeURIComponent(JSON.stringify(payload))))
  const backup = { app: 'mihomo-multipanel', version: 2, exportedAt: new Date().toISOString(), secret: encoded }
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
    const outer = data as { app?: unknown; version?: unknown; profiles?: unknown; settings?: { theme?: unknown; animations?: unknown }; secret?: unknown }
    if (outer.app !== 'mihomo-multipanel') throw new Error('Unsupported backup format.')
    let backup: { profiles?: unknown; settings?: { theme?: unknown; animations?: unknown } }
    if (outer.version === 1) backup = outer
    else if (outer.version === 2 && typeof outer.secret === 'string') {
      backup = JSON.parse(decodeURIComponent(escape(atob(outer.secret))))
    } else throw new Error('Unsupported backup format.')
    if (!Array.isArray(backup.profiles)) throw new Error('Backup contains no valid connection list.')
    const restored = backup.profiles.filter((item): item is Profile => !!item && typeof item === 'object' && typeof (item as Profile).id === 'number' && typeof (item as Profile).name === 'string' && typeof (item as Profile).controller === 'string' && typeof (item as Profile).panelUrl === 'string')
    if (restored.length !== backup.profiles.length) throw new Error('Backup contains invalid connection entries.')
    if (!window.confirm('Import ' + restored.length + ' connection(s) and settings? This will replace current saved data.')) return
    profiles.value = restored
    activeId.value = restored[0]?.id ?? null
    const importedTheme = backup.settings?.theme
    theme.value = ['system', 'light', 'dark'].includes(String(importedTheme)) ? importedTheme as Theme : 'system'
    animationsEnabled.value = backup.settings?.animations !== false
    importMessage.value = 'Backup imported successfully.'
  } catch (error) { importMessage.value = error instanceof Error ? error.message : 'Could not import backup.' }
  finally { input.value = '' }
}
async function checkController() {
  if (!current.value) { controllerStatus.value = 'idle'; return }
  statusRequest?.abort()
  const request = new AbortController()
  statusRequest = request
  const timeout = window.setTimeout(() => request.abort(), 8000)
  controllerStatus.value = 'checking'
  controllerStatusMessage.value = ''
  try {
    const endpoint = current.value.controller.replace(/\/$/, '') + '/version'
    const response = await fetch(endpoint, { signal: request.signal, cache: 'no-store' })
    if (!response.ok) throw new Error('HTTP ' + response.status)
    const data = await response.json()
    controllerStatus.value = 'online'
    controllerStatusMessage.value = typeof data.version === 'string' ? 'Mihomo ' + data.version : 'Controller responds'
    controllerCheckedAt.value = new Date().toLocaleTimeString()
  } catch (error) {
    if (request.signal.aborted && statusRequest !== request) return
    controllerStatus.value = 'offline'
    controllerStatusMessage.value = error instanceof Error && error.name === 'TypeError'
      ? 'Could not reach Controller (network or CORS)'
      : 'Controller did not respond'
    controllerCheckedAt.value = new Date().toLocaleTimeString()
  } finally { window.clearTimeout(timeout) }
}
watch(activeId, () => { if (current.value) checkController() }, { immediate: true })
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

function handleAndroidBack() {
  if (showAbout.value) { showAbout.value = false; return }
  if (showSettings.value) { themeMenuOpen.value = false; showSettings.value = false; return }
  if (showAdd.value) { showAdd.value = false; return }
  if (panelOpen.value) { closePanel(); return }
}
window.addEventListener('android-back', handleAndroidBack)
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
      <div class="hero-top"><h1>One place for<br/><em>every dashboard.</em></h1><button class="settings-button" aria-label="Settings" title="Settings" @click="showSettings = true">⚙</button></div>
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
      <div class="controller-status card" :class="'state-' + controllerStatus">
        <div class="status-row"><span class="status-indicator" :class="controllerStatus"></span><strong>{{ controllerStatus === 'checking' ? 'Checking controller…' : controllerStatus === 'online' ? 'Controller reachable' : controllerStatus === 'offline' ? 'Controller unavailable' : 'Controller status' }}</strong><button class="status-refresh" type="button" :disabled="controllerStatus === 'checking'" aria-label="Check controller again" @click="checkController">↻</button></div>
        <p>{{ controllerStatusMessage || (controllerStatus === 'idle' ? 'Check the connection to Mihomo.' : controllerStatus === 'checking' ? 'Requesting /version…' : '') }}</p>
        <span v-if="controllerCheckedAt" class="status-time">Last check: {{ controllerCheckedAt }}</span>
        <p class="status-disclaimer">A failed check may also be caused by browser network restrictions (CORS).</p>
      </div>
      <div class="preview card"><div class="preview-head"><span class="eyebrow">PANEL URL</span><span class="pill">Ready</span></div><h3>Dashboard</h3><p>{{ current.panelUrl }}</p><button class="open-panel-button" @click="openPanel">Open dashboard <span>↗</span></button><p class="hint">The dashboard opens inside the app. Its own update mechanism remains unchanged.</p></div>
    </section>
    <footer>EARLY PROTOTYPE</footer>
    </template>

    <div v-if="showSettings" class="modal-backdrop" @click.self="showSettings = false">
      <section class="modal card settings-modal">
        <div class="section-heading"><div><span class="eyebrow">PREFERENCES & BACKUP</span><h2>Settings</h2></div><button class="close" aria-label="Close settings" @click="showSettings = false">×</button></div>
        <label for="theme-choice">Appearance</label>
        <div class="theme-picker">
          <button id="theme-choice" class="theme-picker-trigger" type="button" :aria-expanded="themeMenuOpen" aria-haspopup="listbox" @click="themeMenuOpen = !themeMenuOpen">
            <span>{{ themeLabels[theme] }}</span><span class="theme-picker-chevron" :class="{ open: themeMenuOpen }">⌄</span>
          </button>
          <div v-if="themeMenuOpen" class="theme-picker-menu" role="listbox" aria-label="Appearance">
            <button v-for="option in (['system', 'light', 'dark'] as Theme[])" :key="option" class="theme-picker-option" type="button" role="option" :aria-selected="theme === option" @click="theme = option; themeMenuOpen = false">
              <span>{{ themeLabels[option] }}</span><span v-if="theme === option" class="theme-picker-check">✓</span>
            </button>
          </div>
        </div>
        <div class="animation-setting">
          <div class="animation-setting-copy"><strong>Interface animations</strong><span>Subtle transitions and button feedback</span></div>
          <button class="switch" type="button" role="switch" :aria-checked="animationsEnabled" :class="{ enabled: animationsEnabled }" @click="animationsEnabled = !animationsEnabled"><span></span></button>
        </div>
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
        <div class="about-brand"><div class="about-logo" aria-label="Mihomo MultiPanel app icon"><svg viewBox="0 0 512 512" role="img" aria-hidden="true"><g transform="translate(92 92) scale(.64)"><path fill="currentColor" d="M256 64 432 160 432 352 256 448 80 352 80 160Z"/><g fill="none" stroke="#fff" stroke-width="16" stroke-linecap="round" stroke-linejoin="round"><path d="M256 64 432 160 432 352 256 448 80 352 80 160Z"/></g></g></svg></div><div><strong>Mihomo MultiPanel</strong><span>Version 0.1.0</span></div></div>
        <p class="hint about-description">A simple hub for managing Mihomo controllers and web dashboards.</p>
        <a class="about-link" href="https://github.com/kwerty2897/mihomo-multipanel" target="_blank" rel="noreferrer">GitHub project <span>↗</span></a>
        <p class="about-footer">Made for the Mihomo community.</p>
      </section>
    </div>

    <div v-if="showAdd" class="modal-backdrop" @click.self="showAdd = false">
      <section class="modal card">
        <div class="section-heading"><div><span class="eyebrow">NEW CONNECTION</span><h2>Add controller</h2></div><button class="close" @click="showAdd = false">×</button></div>
        <label for="profile-name">Name (optional)</label><input id="profile-name" v-model="name" placeholder="Home, VPS, Router…"/>
        <label for="connection-url">Dashboard / controller URL</label><input id="connection-url" :value="connectionUrl" @input="connectionUrl = ($event.target as HTMLInputElement).value.toLowerCase()" placeholder="http://192.168.1.1:9090/ui/metacubexd" autocapitalize="none" spellcheck="false"/>
        <p class="hint">Enter the dashboard URL. The controller address is detected automatically from its origin. Input is converted to lowercase.</p>
        <div class="modal-actions"><button class="secondary" @click="showAdd = false">Cancel</button><button @click="addProfile" :disabled="!normalizeUrl(connectionUrl)">Save</button></div>
      </section>
    </div>
  </main>
</template>
