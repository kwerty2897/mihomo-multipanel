<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type Profile = { id: number; name: string; controller: string; panelUrl: string; panelTemplateId?: string; host?: string; port?: string; secret?: string; mihomoUrl?: string }
type PanelTemplate = { id: string; name: string; url: string }
const DEFAULT_TEMPLATES: PanelTemplate[] = [{ id: 'yacd', name: 'YACD', url: 'https://yacd.metacubex.one/?hostname=%host&port=%port&secret=%secret' }, { id: 'zash', name: 'Zash', url: 'https://board.zash.run.place/#/setup?http=true&hostname=%host&port=%port&secret=%secret' }, { id: 'metacubexd', name: 'MetaCubeXD', url: 'https://metacubex.github.io/metacubexd/#/setup?http=true&hostname=%host&port=%port&secret=%secret' }]
const STORAGE_KEY = 'mihomo-multipanel-profiles-v1'
const SETTINGS_KEY = 'mihomo-multipanel-settings-v1'
type Theme = 'system' | 'light' | 'dark'
const savedSettings = (() => {
  try { return JSON.parse(localStorage.getItem(SETTINGS_KEY) || '{}') } catch { return {} }
})()
const theme = ref<Theme>(['system', 'light', 'dark'].includes(savedSettings.theme) ? savedSettings.theme : 'system')
const animationsEnabled = ref(savedSettings.animations !== false)
const hidePanelToolbar = ref(savedSettings.hidePanelToolbar === true)
const panelTemplates = ref<PanelTemplate[]>(Array.isArray(savedSettings.panelTemplates) && savedSettings.panelTemplates.length ? savedSettings.panelTemplates : DEFAULT_TEMPLATES)
const selectedTemplateId = ref('')
const templatePickerOpen = ref(false)
const revealSecret = ref(false)
const templateName = ref('')
const templateUrl = ref('')
const editingTemplateId = ref('')
watch([hidePanelToolbar, panelTemplates], ([value, templates]) => { try { const current = JSON.parse(localStorage.getItem(SETTINGS_KEY) || '{}'); localStorage.setItem(SETTINGS_KEY, JSON.stringify({ ...current, hidePanelToolbar: value, panelTemplates: templates })) } catch { /* storage may be unavailable */ } }, { deep: true })
const panelOpen = ref(false)
const panelLoading = ref(false)
const showSettings = ref(false)
const showAbout = ref(false)
watch(showSettings, value => {
  document.body.style.overflow = value ? 'hidden' : ''
  document.body.style.touchAction = value ? 'none' : ''
})

const themeMenuOpen = ref(false)
const themeLabels: Record<Theme, string> = { system: 'Use system theme', light: 'Light', dark: 'Dark' }
const greeting = computed(() => { const hour = new Date().getHours(); return hour < 12 ? 'morning' : hour < 18 ? 'afternoon' : 'evening' })
const importInput = ref<HTMLInputElement | null>(null)
const importMessage = ref('')
watch([theme, animationsEnabled], ([themeValue, animationsValue]) => {
  try { localStorage.setItem(SETTINGS_KEY, JSON.stringify({ theme: themeValue, animations: animationsValue, hidePanelToolbar: hidePanelToolbar.value, panelTemplates: panelTemplates.value })) } catch { /* storage may be unavailable */ }
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
const showEdit = ref(false)
const editingId = ref<number | null>(null)
const name = ref('')
const connectionUrl = ref('')
const host = ref('')
const port = ref('9090')
const secret = ref('')
const mihomoUrl = ref('')
const manualPanelUrl = ref('')
const editTemplatePickerOpen = ref(false)
let longPressTimer: number | null = null
function startLongPress(profile: Profile) { cancelLongPress(); longPressTimer = window.setTimeout(() => { editingId.value = profile.id; name.value = profile.name; host.value = profile.host || (() => { try { return new URL(profile.controller).hostname } catch { return '' } })(); port.value = profile.port || (() => { try { return new URL(profile.controller).port || '9090' } catch { return '9090' } })(); secret.value = profile.secret || ''; mihomoUrl.value = profile.mihomoUrl || ''; selectedTemplateId.value = profile.panelTemplateId || ''; manualPanelUrl.value = profile.panelTemplateId ? '' : profile.panelUrl; revealSecret.value = false; showEdit.value = true }, 550) }
function cancelLongPress() { if (longPressTimer !== null) window.clearTimeout(longPressTimer); longPressTimer = null }
function scrollInputIntoView(event: FocusEvent) { window.setTimeout(() => (event.target as HTMLElement)?.scrollIntoView({ block: 'center', behavior: 'smooth' }), 250) }
function renderTemplate(template: string) { return template.replace(/%host/g, encodeURIComponent(host.value.trim())).replace(/%port/g, encodeURIComponent(port.value.trim())).replace(/%secret/g, encodeURIComponent(secret.value)) }
function controllerFromFields() { const h = host.value.trim().replace(/^https?:\/\//, '').replace(/\/$/, ''); const pt = port.value.trim(); if (!h || !/^\d{1,5}$/.test(pt) || Number(pt) < 1 || Number(pt) > 65535) return ''; return normalizeUrl('http://' + h + ':' + pt) }
function deriveController(value: string) { const normalized = normalizeUrl(value); if (!normalized) return ''; try { return new URL(normalized).origin } catch { return '' } }
const current = computed(() => profiles.value.find(p => p.id === activeId.value) ?? null)
function normalizeUrl(value: string) {
  const v = value.trim()
  if (!v) return ''
  try { const u = new URL(v.includes('://') ? v : 'http://' + v); if (!['http:', 'https:'].includes(u.protocol)) return ''; return u.toString().replace(/\/$/, '') } catch { return '' }
}
function saveEdit() {
  const controller = controllerFromFields()
  const selected = panelTemplates.value.find(t => t.id === selectedTemplateId.value)
  const panel = normalizeUrl(selected ? renderTemplate(selected.url) : manualPanelUrl.value)
  if (!controller || !panel || editingId.value === null) return
  
  const index = profiles.value.findIndex(p => p.id === editingId.value)
  if (index < 0) return
  profiles.value[index] = { ...profiles.value[index], name: name.value.trim() || controller, controller: normalizeUrl(mihomoUrl.value) || controller, panelUrl: panel, panelTemplateId: selected?.id || '', host: host.value.trim(), port: port.value.trim(), secret: secret.value, mihomoUrl: normalizeUrl(mihomoUrl.value) || '' }
  showEdit.value = false
  editingId.value = null
}
function addProfile() {
  const selected = panelTemplates.value.find(t => t.id === selectedTemplateId.value)
  const p = normalizeUrl(selected ? renderTemplate(selected.url) : manualPanelUrl.value)
  const c = controllerFromFields()
  if (!c || !p) return
  const parsed = (() => { try { return new URL(c) } catch { return null } })(); const item: Profile = { id: Date.now(), name: name.value.trim() || c, controller: normalizeUrl(mihomoUrl.value) || c, panelUrl: p, panelTemplateId: selected?.id || '', host: host.value.trim() || parsed?.hostname || '', port: port.value.trim() || parsed?.port || '9090', secret: secret.value, mihomoUrl: normalizeUrl(mihomoUrl.value) || '' }
  profiles.value.push(item); activeId.value = item.id
  name.value = ''; host.value = ''; port.value = '9090'; secret.value = ''; mihomoUrl.value = ''; manualPanelUrl.value = ''; selectedTemplateId.value = ''; connectionUrl.value = ''; showAdd.value = false
}
function exportSettings() {
  const payload = { settings: { theme: theme.value, animations: animationsEnabled.value, hidePanelToolbar: hidePanelToolbar.value, panelTemplates: panelTemplates.value }, profiles: profiles.value }
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
    const outer = data as { app?: unknown; version?: unknown; profiles?: unknown; settings?: { theme?: unknown; animations?: unknown; hidePanelToolbar?: unknown; panelTemplates?: unknown }; secret?: unknown }
    if (outer.app !== 'mihomo-multipanel') throw new Error('Unsupported backup format.')
    let backup: { profiles?: unknown; settings?: { theme?: unknown; animations?: unknown; hidePanelToolbar?: unknown; panelTemplates?: unknown } }
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
    hidePanelToolbar.value = backup.settings?.hidePanelToolbar === true
    panelTemplates.value = Array.isArray(backup.settings?.panelTemplates) && backup.settings!.panelTemplates!.length ? backup.settings!.panelTemplates as PanelTemplate[] : DEFAULT_TEMPLATES
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
function saveTemplate() { const n = templateName.value.trim(); const u = templateUrl.value.trim(); if (!n || !u) return; if (editingTemplateId.value) { const i = panelTemplates.value.findIndex(t => t.id === editingTemplateId.value); if (i >= 0) panelTemplates.value[i] = { id: editingTemplateId.value, name: n, url: u }; } else panelTemplates.value.push({ id: 'custom-' + Date.now(), name: n, url: u }); templateName.value = ''; templateUrl.value = ''; editingTemplateId.value = '' }
function editTemplate(t: PanelTemplate) { editingTemplateId.value = t.id; templateName.value = t.name; templateUrl.value = t.url }
function deleteTemplate(id: string) { panelTemplates.value = panelTemplates.value.filter(t => t.id !== id); if (selectedTemplateId.value === id) selectedTemplateId.value = panelTemplates.value[0]?.id || '' }
function removeProfile(id: number) {
  profiles.value = profiles.value.filter(p => p.id !== id)
  if (activeId.value === id) activeId.value = profiles.value[0]?.id ?? null
}

function handleAndroidBack() {
  if (showAbout.value) { showAbout.value = false; return }
  if (showSettings.value) { themeMenuOpen.value = false; showSettings.value = false; return }
  if (showEdit.value) { showEdit.value = false; return }
  if (showAdd.value) { showAdd.value = false; return }
  if (panelOpen.value) { closePanel(); return }
}
window.addEventListener('android-back', handleAndroidBack)
</script>

<template>
  <main class="shell">
    <section v-if="panelOpen && current" class="panel-viewer">
      <header v-if="!hidePanelToolbar" class="panel-toolbar">
        <button class="panel-toolbar-button" aria-label="Back to connections" @click="closePanel">‹</button>
        <div class="panel-toolbar-title"><strong>{{ current.name }}</strong><span>{{ current.panelUrl }}</span></div>
        <button class="panel-toolbar-button" aria-label="Reload panel" @click="reloadPanel">↻</button>
      </header>
      <div v-if="panelLoading" class="panel-loading" :class="{ toolbarHidden: hidePanelToolbar }">Loading dashboard…</div>
      <iframe :key="current.id" class="panel-frame" :src="current.panelUrl" :title="current.name + ' dashboard'" allow="clipboard-read; clipboard-write; fullscreen" allowfullscreen @load="panelLoading = false"></iframe>
    </section>
    <template v-else>
    <section id="overview" class="hero">
      <div class="hero-top"><h1>Good <em>{{ greeting }}.</em></h1><button class="settings-button" aria-label="Settings" title="Settings" @click="showSettings = true">⚙</button></div>
    </section>
    <section id="connections" class="connection card">
      <div class="section-heading"><div><span class="eyebrow">CONTROLLERS</span><h2>Your connections</h2></div><span class="pill">{{ profiles.length }} saved</span></div>
      <div v-if="profiles.length" class="panel-list">
        <div v-for="profile in profiles" :key="profile.id" class="panel-card" :class="{selected: activeId === profile.id}" role="button" tabindex="0" @click="activeId = profile.id" @keydown.enter="activeId = profile.id" @pointerdown="startLongPress(profile)" @pointerup="cancelLongPress" @pointerleave="cancelLongPress" @pointercancel="cancelLongPress" @contextmenu.prevent="startLongPress(profile)">
          <div class="panel-copy"><strong>{{ profile.name }}</strong><span>{{ profile.controller }}</span></div><div class="arrow">›</div>
        </div>
      </div>
      <div v-else class="empty">No saved connections yet. Add your Mihomo controller and dashboard URL.</div>
      <button class="add-button" @click="host = '127.0.0.1'; port = '9090'; selectedTemplateId = ''; secret = ''; mihomoUrl = ''; manualPanelUrl = ''; showAdd = true"><span class="plus">＋</span> Add connection</button>
    </section>
    <section v-if="current" id="dashboard" class="panels">
      <div class="section-heading"><div><span class="eyebrow">DASHBOARD</span><h2>{{ current.name }}</h2></div></div>
      <div class="preview card"><div class="preview-head"><span class="eyebrow">PANEL URL</span><span class="pill">Ready</span></div><h3>Dashboard</h3><p>{{ current.panelUrl }}</p><button class="open-panel-button" @click="openPanel">Open dashboard <span>↗</span></button><p class="hint">The dashboard opens inside the app. Its own update mechanism remains unchanged.</p></div>
    </section>
    <footer>EARLY PROTOTYPE</footer>
    </template>

    <Transition name="settings-exit">
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
        <div class="animation-setting">
          <div class="animation-setting-copy"><strong>Hide dashboard toolbar</strong><span>Hide the in-app back and reload buttons while viewing a panel</span></div>
          <button class="switch" type="button" role="switch" :aria-checked="hidePanelToolbar" :class="{ enabled: hidePanelToolbar }" @click="hidePanelToolbar = !hidePanelToolbar"><span></span></button>
        </div>
        <div class="settings-divider"></div>
        <div class="backup-title">Dashboard templates</div>
        <p class="hint">Templates use %host, %port and %secret placeholders. Choose a template when adding a connection.</p>
        <div v-for="t in panelTemplates" :key="t.id" class="animation-setting template-row"><div class="animation-setting-copy"><strong>{{ t.name }}</strong><span>{{ t.url }}</span></div><button class="secondary" @click="editTemplate(t)">Edit</button><button class="secondary" @click="deleteTemplate(t.id)">Remove</button></div>
        <label for="template-name">Template name</label><input id="template-name" v-model="templateName" placeholder="My dashboard"/><label for="template-url">Template URL</label><textarea id="template-url" v-model="templateUrl" rows="3" placeholder="https://example.com/?hostname=%host&port=%port&secret=%secret"></textarea><div class="modal-actions"><button class="secondary" @click="editingTemplateId='';templateName='';templateUrl=''">Clear</button><button @click="saveTemplate" :disabled="!templateName.trim() || !templateUrl.trim()">{{ editingTemplateId ? 'Update template' : 'Add template' }}</button></div>
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
    </Transition>

    <div v-if="showAbout" class="modal-backdrop about-backdrop" @click.self="showAbout = false">
      <section class="modal card about-modal">
        <div class="section-heading"><div><span class="eyebrow">APPLICATION</span><h2>About</h2></div><button class="close" aria-label="Close about" @click="showAbout = false">×</button></div>
        <div class="about-brand"><div class="about-logo" aria-label="Mihomo MultiPanel app icon"><svg viewBox="0 0 512 512" role="img" aria-hidden="true"><g transform="translate(92 92) scale(.64)"><path fill="currentColor" d="M256 64 432 160 432 352 256 448 80 352 80 160Z"/><g fill="none" stroke="#fff" stroke-width="16" stroke-linecap="round" stroke-linejoin="round"><path d="M256 64 432 160 432 352 256 448 80 352 80 160Z"/></g></g></svg></div><div><strong>Mihomo MultiPanel</strong><span>Version 0.1.0</span></div></div>
        <p class="hint about-description">A simple hub for managing Mihomo controllers and web dashboards.</p>
        <a class="about-link" href="https://github.com/kwerty2897/mihomo-multipanel" target="_blank" rel="noreferrer">GitHub project <span>↗</span></a>
        <p class="about-footer">Made for the Mihomo community.</p>
      </section>
    </div>

    <div v-if="showEdit" class="modal-backdrop" @click.self="showEdit = false">
      <section class="modal card">
        <div class="section-heading"><div><span class="eyebrow">CONNECTION</span><h2>Edit controller</h2></div><button class="close" @click="showEdit = false">×</button></div>
        <label for="edit-profile-name">Name (optional)</label><input id="edit-profile-name" v-model="name" placeholder="Home, VPS, Router…"/>

        <template v-if="selectedTemplateId"><label for="edit-host">Host / IP</label><input id="edit-host" v-model="host" placeholder="192.168.1.1" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><label for="edit-port">Port</label><input id="edit-port" v-model="port" inputmode="numeric" placeholder="9090" @focus="scrollInputIntoView"/><label for="edit-secret">Secret (optional)</label><div class="secret-input-wrap"><input id="edit-secret" v-model="secret" :type="revealSecret ? 'text' : 'password'" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><button type="button" class="secret-toggle" @click="revealSecret = !revealSecret">{{ revealSecret ? 'Hide' : 'Show' }}</button></div><label for="edit-mihomo-url">Mihomo URL (optional)</label><input id="edit-mihomo-url" v-model="mihomoUrl" placeholder="http://192.168.1.1:9090" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><p class="hint">Use this if Mihomo is managed through another dashboard or its API is exposed at a different address.</p></template><template v-else><label for="edit-manual-panel-url">Dashboard URL</label><input id="edit-manual-panel-url" v-model="manualPanelUrl" placeholder="https://dashboard.example" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><p class="hint">Configure the connection manually inside the dashboard.</p></template>
                <label>Dashboard template</label><div class="template-picker"><button type="button" class="template-picker-trigger" :aria-expanded="editTemplatePickerOpen" @click="editTemplatePickerOpen = !editTemplatePickerOpen"><span>{{ panelTemplates.find(t => t.id === selectedTemplateId)?.name || 'Not selected' }}</span><span class="picker-chevron">⌄</span></button><div v-if="editTemplatePickerOpen" class="template-picker-menu"><button type="button" class="template-picker-option" @click="selectedTemplateId = ''; editTemplatePickerOpen = false">Not selected</button><button v-for="t in panelTemplates" :key="t.id" type="button" class="template-picker-option" @click="selectedTemplateId = t.id; editTemplatePickerOpen = false">{{ t.name }}</button></div></div>
        <div class="modal-actions"><button class="secondary" @click="showEdit = false">Cancel</button><button @click="saveEdit" :disabled="!controllerFromFields() || (selectedTemplateId ? false : !normalizeUrl(manualPanelUrl))">Save</button><button class="secondary" @click="removeProfile(editingId!); showEdit = false">Delete</button></div>
      </section>
    </div>

    <div v-if="showAdd" class="modal-backdrop" @click.self="showAdd = false">
      <section class="modal card">
        <div class="section-heading"><div><span class="eyebrow">NEW CONNECTION</span><h2>Add controller</h2></div><button class="close" @click="showAdd = false">×</button></div>
        <label for="profile-name">Name (optional)</label><input id="profile-name" v-model="name" placeholder="Home, VPS, Router…"/>
        <label>Dashboard template</label><div class="template-picker"><button type="button" class="template-picker-trigger" :aria-expanded="templatePickerOpen" @click="templatePickerOpen = !templatePickerOpen"><span>{{ panelTemplates.find(t => t.id === selectedTemplateId)?.name || 'Not selected' }}</span><span class="picker-chevron">⌄</span></button><div v-if="templatePickerOpen" class="template-picker-menu"><button type="button" class="template-picker-option" @click="selectedTemplateId = ''; templatePickerOpen = false">Not selected</button><button v-for="t in panelTemplates" :key="t.id" type="button" class="template-picker-option" @click="selectedTemplateId = t.id; templatePickerOpen = false">{{ t.name }}</button></div></div>
        <template v-if="selectedTemplateId"><label for="add-host">Host / IP</label><input id="add-host" v-model="host" placeholder="192.168.1.1" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><label for="add-port">Port</label><input id="add-port" v-model="port" inputmode="numeric" placeholder="9090" @focus="scrollInputIntoView"/><label for="add-secret">Secret (optional)</label><div class="secret-input-wrap"><input id="add-secret" v-model="secret" :type="revealSecret ? 'text' : 'password'" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><button type="button" class="secret-toggle" @click="revealSecret = !revealSecret">{{ revealSecret ? 'Hide' : 'Show' }}</button></div><label for="add-mihomo-url">Mihomo URL (optional)</label><input id="add-mihomo-url" v-model="mihomoUrl" placeholder="http://192.168.1.1:9090" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><p class="hint">Optional direct API address, used for availability checks if the dashboard is hosted elsewhere.</p></template>
        <template v-else><label for="manual-panel-url">Dashboard URL</label><input id="manual-panel-url" v-model="manualPanelUrl" placeholder="https://dashboard.example" autocapitalize="none" spellcheck="false" @focus="scrollInputIntoView"/><p class="hint">The dashboard will open as entered. Configure its connection manually inside the dashboard.</p></template>
        <div class="modal-actions"><button class="secondary" @click="showAdd = false">Cancel</button><button @click="addProfile" :disabled="!host.trim() || (selectedTemplateId ? !controllerFromFields() : !normalizeUrl(manualPanelUrl))">Save</button></div>
      </section>
    </div>
  </main>
</template>
