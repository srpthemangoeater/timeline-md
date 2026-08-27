<script setup>
import { computed, nextTick, ref, watch } from 'vue'
import { toPng } from 'html-to-image'
import { ChevronDown, Download, FileText, Filter, HelpCircle, History, LayoutTemplate, Maximize2, Minimize2, Plus, RotateCcw, Sparkles, Upload } from 'lucide-vue-next'

const page = ref('editor')
const syntaxGuideOpen = ref(true)
const previewWidthMode = ref('comfortable')
const hiddenLanes = ref([])
const filterFrom = ref('')
const filterTo = ref('')
const filtersOpen = ref(false)
function isSet(v) { return v !== '' && v !== null && v !== undefined && !Number.isNaN(Number(v)) }
function toggleLane(name) {
  hiddenLanes.value = hiddenLanes.value.includes(name) ? hiddenLanes.value.filter(n => n !== name) : [...hiddenLanes.value, name]
}
function resetFilters() { hiddenLanes.value = []; filterFrom.value = ''; filterTo.value = '' }
const editorWidth = ref(420)
let resizing = false
function startResize() {
  resizing = true
  document.body.style.cursor = 'col-resize'
  document.body.style.userSelect = 'none'
  window.addEventListener('mousemove', onResize)
  window.addEventListener('mouseup', stopResize)
}
function onResize(e) {
  if (!resizing) return
  editorWidth.value = Math.min(720, Math.max(280, e.clientX))
}
function stopResize() {
  resizing = false
  document.body.style.cursor = ''
  document.body.style.userSelect = ''
  window.removeEventListener('mousemove', onResize)
  window.removeEventListener('mouseup', stopResize)
}
const changelog = [
  {
    version: '1.3.0',
    date: '2026-08-27',
    changes: [
      'Fixed overlapping event captions and connector arrows in the timeline preview.',
      'Reworked lane spacing so labels, captions, and tracks no longer collide.',
      'Added a Changelog tab.',
    ],
  },
  {
    version: '1.2.0',
    date: '2026-08-26',
    changes: [
      'Removed the local Save button — timelines now save to your browser automatically.',
      'Repositioned the markdown and preview action buttons.',
      'Kept right-edge events inside the visible preview area.',
    ],
  },
  {
    version: '1.1.0',
    date: '2026-08-25',
    changes: [
      'Polished lane alignment and event label readability.',
      'Improved event readability and lane start markers.',
      'Refined the overall timeline editor preview layout.',
    ],
  },
  {
    version: '1.0.0',
    date: '2026-08-24',
    changes: [
      'Initial release: write Markdown, see it rendered as a visual timeline.',
      'Support for lanes, colors, linked events, and PNG export.',
    ],
  },
]

const starter = `# The Nameless Day

> Twenty years ago, one day vanished from every memory.

## AKENEDIA | #f5a6ad
- [-300] A1: The last clear morning > The palace bells ring at dawn.
- [+80] A2: The sky fractures > A white seam appears above the capital.

## LOWER CITY | #ebc2d7
- [-300] B1: The first witness > A courier sees the light descend.
- [+720] B2: The missing hour > Every clock stops at once.

## HOLLOWED GROUNDS | #adc4f2
- [+80] C1: Lume awakens > The mines begin to sing.
- [+720] C2: The memory ends > No record survives beyond this point.

@link A1 -> B1
@link A2 -> C1
@link B2 -> C2`

const markdown = ref(localStorage.getItem('timeline-md-content') || starter)
const fileName = ref(localStorage.getItem('timeline-md-name') || 'product-launch.md')
const notice = ref('')
const timelineEl = ref(null)

const parsed = computed(() => parseMarkdown(markdown.value))
const dataRange = computed(() => {
  const values = parsed.value.lanes.flatMap(lane => lane.events.map(event => event.position))
  if (!values.length) return { min: -4, max: 16 }
  const min = Math.min(...values), max = Math.max(...values)
  return { min: Math.min(0, min), max: Math.max(10, max) }
})
const range = computed(() => ({
  min: isSet(filterFrom.value) ? Number(filterFrom.value) : dataRange.value.min,
  max: isSet(filterTo.value) ? Number(filterTo.value) : dataRange.value.max,
}))
const filtersActive = computed(() => hiddenLanes.value.length > 0 || isSet(filterFrom.value) || isSet(filterTo.value))
const visibleLanes = computed(() => parsed.value.lanes
  .filter(lane => !hiddenLanes.value.includes(lane.name))
  .map(lane => ({ ...lane, events: lane.events.filter(event => event.position >= range.value.min && event.position <= range.value.max) })))
const ticks = computed(() => {
  const span = range.value.max - range.value.min
  const step = span <= 24 ? 2 : span <= 80 ? 10 : span <= 240 ? 25 : span <= 600 ? 100 : 200
  const result = []
  const first = Math.ceil(range.value.min / step) * step
  for (let i = first; i <= range.value.max; i += step) result.push(i)
  return result
})
const saved = computed(() => localStorage.getItem('timeline-md-content') === markdown.value)

function parseMarkdown(source) {
  const lines = source.split(/\r?\n/)
  const title = (lines.find(line => /^# /.test(line)) || '# Untitled timeline').replace(/^# /, '').trim()
  const subtitle = (lines.find(line => /^> /.test(line)) || '').replace(/^> /, '').trim()
  const lanes = []
  const links = []
  let lane = null
  for (const line of lines) {
    const heading = line.match(/^##\s+(.+)/)
    if (heading) {
      const parts = heading[1].split('|')
      lane = { name: parts[0].trim().replace(/\s+—\s+/g, ' / '), color: /^\s*#(?:[0-9a-f]{3}){1,2}\s*$/i.test(parts[1] || '') ? parts[1].trim() : '#f5a6ad', events: [] }
      lanes.push(lane); continue
    }
    const link = line.match(/^\s*@link\s+(\S+)\s*->\s*(\S+)/i)
    if (link) { links.push({ from: link[1], to: link[2] }); continue }
    const event = line.match(/^\s*-\s+\[([+-]?\d+(?:\.\d+)?)\]\s+(\S+?)(?::\s*|\s+)(.+?)\s*$/) || line.match(/^\s*-\s+(?:\[([^\]]+)\]\s*)?(.+?)(?:\s+@\s*(-?\d+(?:\.\d+)?))?\s*$/)
    if (event) {
      if (!lane) { lane = { name: 'Timeline', events: [] }; lanes.push(lane) }
      const modern = event[3] !== undefined && /^[+-]?\d/.test(event[1])
      lane.events.push(modern
        ? { id: event[2], label: event[3], position: Number(event[1]) }
        : { id: event[1] || `E${lane.events.length + 1}`, label: event[2], position: Number(event[3] ?? lane.events.length * 2) })
    }
  }
  return { title, subtitle, links, lanes: lanes.length ? lanes : [{ name: 'Timeline', color: '#f5a6ad', events: [] }] }
}
function position(value) { return ((value - range.value.min) / (range.value.max - range.value.min)) * 100 }

// Must match the lane layout constants in style.css (label -> gap -> track -> caption zone -> gutter).
const LANE_LABEL_H = 24
const LANE_LABEL_GAP = 10
const LANE_TRACK_H = 20
const LANE_ZONE_H = 84
const LANE_GUTTER_H = 56
const LANE_ROW_H = LANE_LABEL_H + LANE_LABEL_GAP + LANE_TRACK_H + LANE_ZONE_H + LANE_GUTTER_H
const LANE_TRACK_CENTER = LANE_LABEL_H + LANE_LABEL_GAP + LANE_TRACK_H / 2

const canvasWidth = computed(() => {
  if (previewWidthMode.value === 'fit') return '100%'
  const eventCount = visibleLanes.value.reduce((n, lane) => n + lane.events.length, 0)
  return Math.max(720, ticks.value.length * 90, eventCount * 130) + 'px'
})

const connectors = computed(() => {
  const lanes = visibleLanes.value
  const laneIndexOf = id => lanes.findIndex(lane => lane.events.some(event => event.id === id))
  const eventById = id => { for (const lane of lanes) { const found = lane.events.find(event => event.id === id); if (found) return found } return null }
  return parsed.value.links.map(link => {
    const fromIndex = laneIndexOf(link.from)
    const toIndex = laneIndexOf(link.to)
    const fromEvent = eventById(link.from)
    if (fromIndex === -1 || toIndex === -1 || toIndex <= fromIndex || !fromEvent) return null
    const top = fromIndex * LANE_ROW_H + LANE_TRACK_CENTER
    const bottom = toIndex * LANE_ROW_H + LANE_TRACK_CENTER
    return { id: `${link.from}-${link.to}`, left: position(fromEvent.position), top, height: bottom - top }
  }).filter(Boolean)
})

function laneStart(lane) { return lane.events.length ? Math.min(...lane.events.map(event => event.position)) : range.value.min }
function lanePosition(value, lane) {
  const start = laneStart(lane)
  return ((value - start) / (range.value.max - start || 1)) * 100
}
function eventTitle(label) { return label.split(/\s+—\s+|\s+>\s+/)[0] }
function eventDetail(label) { return label.split(/\s+—\s+|\s+>\s+/).slice(1).join(' > ') }
function eventEdge(event, lane) {
  const value = lanePosition(event.position, lane)
  return value > 82 ? 'edge-right' : value < 12 ? 'edge-left' : ''
}
function save() { localStorage.setItem('timeline-md-content', markdown.value); localStorage.setItem('timeline-md-name', fileName.value); notice.value = 'Saved locally'; setTimeout(() => notice.value = '', 1800) }
function reset() { markdown.value = starter; fileName.value = 'product-launch.md'; save() }
function newTimeline() { markdown.value = '# New timeline\n\n## Act 1 — Start\n- [A1] First event @ 0'; fileName.value = 'untitled-timeline.md'; page.value = 'editor' }
function loadFile(event) {
  const file = event.target.files?.[0]; if (!file) return
  const reader = new FileReader(); reader.onload = () => { markdown.value = reader.result; fileName.value = file.name; page.value = 'editor' }; reader.readAsText(file)
}
function exportMarkdown() {
  const blob = new Blob([markdown.value], { type: 'text/markdown;charset=utf-8' })
  const link = document.createElement('a'); link.download = fileName.value || 'timeline.md'; link.href = URL.createObjectURL(blob); link.click(); URL.revokeObjectURL(link.href)
}
async function exportPng(scope = 'view') {
  if (!timelineEl.value) return
  let snapshot = null
  if (scope === 'full' && filtersActive.value) {
    snapshot = { hidden: hiddenLanes.value, from: filterFrom.value, to: filterTo.value }
    hiddenLanes.value = []; filterFrom.value = ''; filterTo.value = ''
    await nextTick()
  }
  try { const dataUrl = await toPng(timelineEl.value, { pixelRatio: 2, backgroundColor: '#f8fafc' }); const link = document.createElement('a'); link.download = `${fileName.value.replace(/\.md$/i, '')}.png`; link.href = dataUrl; link.click(); notice.value = 'PNG exported'; setTimeout(() => notice.value = '', 1800) } catch { notice.value = 'Export failed' }
  if (snapshot) { hiddenLanes.value = snapshot.hidden; filterFrom.value = snapshot.from; filterTo.value = snapshot.to; await nextTick() }
}
watch(markdown, () => { localStorage.setItem('timeline-md-draft', markdown.value) })
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand"><div class="brand-mark"><Sparkles :size="17" /></div><span>Timeline<span class="brand-accent">.md</span></span></div>
      <nav class="page-nav">
        <button :class="{ selected: page === 'editor' }" @click="page = 'editor'"><LayoutTemplate :size="15" /> Timeline Editor</button>
        <button :class="{ selected: page === 'changelog' }" @click="page = 'changelog'"><History :size="15" /> Changelog</button>
      </nav>
      <div class="header-actions"><span v-if="notice" class="notice">{{ notice }}</span><button class="icon-btn" title="Help"><HelpCircle :size="18" /></button><button class="avatar">M</button></div>
    </header>
    <main v-if="page === 'editor'" class="workspace">
      <aside class="sidebar">
        <div class="sidebar-title"><span>MY TIMELINES</span><button class="small-icon" @click="newTimeline"><Plus :size="16" /></button></div>
        <button class="timeline-file active"><FileText :size="16" /><span>{{ fileName }}</span><span v-if="!saved" class="unsaved">•</span></button>
        <div class="sidebar-footer"><p>Everything is saved in your browser.</p><button class="reset-btn" @click="reset"><RotateCcw :size="14" /> Reset example</button></div>
      </aside>
      <section class="content">
        <div class="page-heading"><div><p class="eyebrow">VISUAL PLANNING TOOL</p><h1>Build your timeline<span class="heading-dot">.</span></h1><p class="subheading">Write simple Markdown. See your story unfold.</p></div><div class="page-actions"><label class="btn secondary"><Upload :size="16" /> Import .md<input type="file" accept=".md,.markdown,text/markdown" @change="loadFile" hidden /></label><button class="btn primary" @click="exportPng"><Download :size="16" /> Export PNG</button></div></div>
        <div class="editor-pane" :style="{ flexBasis: editorWidth + 'px' }">
          <div class="editor-card"><div class="editor-head"><div class="source-heading"><strong>Story source</strong><span class="language-pill">MARKDOWN</span><label class="inline-action">Import .md<input type="file" accept=".md,.markdown,text/markdown" @change="loadFile" hidden /></label><button class="inline-action" @click="exportMarkdown">Export .md</button></div></div><textarea v-model="markdown" spellcheck="false" aria-label="Markdown timeline source"></textarea><div class="syntax-guide"><button class="syntax-guide-toggle" @click="syntaxGuideOpen = !syntaxGuideOpen"><ChevronDown :size="14" :class="{ collapsed: !syntaxGuideOpen }" /> Quick syntax guide</button><pre v-show="syntaxGuideOpen" class="syntax-guide-body"># Title
&gt; Subtitle
## Lane | #hexcolor
- [-300] ID: Event &gt; Note
- [+450] ID: Event &gt; Note
@link ID1 -&gt; ID2</pre></div></div>
        </div>
        <div class="resizer" @mousedown="startResize"></div>
        <div class="preview-pane">
          <div class="preview-heading">
            <div><h2>Live preview</h2><p>{{ visibleLanes.reduce((n, l) => n + l.events.length, 0) }} events · positions relative to first event</p></div>
            <div class="preview-actions">
              <div class="range-label">{{ range.min }} <span>→</span> {{ range.max }} <small>units</small></div>
              <button class="ghost-toggle" :class="{ active: filtersOpen || filtersActive }" @click="filtersOpen = !filtersOpen"><Filter :size="13" /> Filters<span v-if="filtersActive" class="filter-dot"></span></button>
              <div class="width-toggle">
                <button :class="{ selected: previewWidthMode === 'fit' }" title="Fit to screen" @click="previewWidthMode = 'fit'"><Minimize2 :size="13" /></button>
                <button :class="{ selected: previewWidthMode === 'comfortable' }" title="Comfortable spacing" @click="previewWidthMode = 'comfortable'"><Maximize2 :size="13" /></button>
              </div>
              <button v-if="filtersActive" class="preview-export ghost" @click="exportPng('full')">Export full</button>
              <button class="preview-export" @click="exportPng('view')"><Download :size="14" /> Export PNG</button>
            </div>
          </div>
          <div v-if="filtersOpen" class="filters-panel">
            <div class="filter-group">
              <span class="filter-label">Lanes</span>
              <div class="lane-chips">
                <button v-for="lane in parsed.lanes" :key="lane.name" class="lane-chip" :class="{ off: hiddenLanes.includes(lane.name) }" @click="toggleLane(lane.name)"><span class="chip-dot" :style="{ backgroundColor: lane.color }"></span>{{ lane.name }}</button>
              </div>
            </div>
            <div class="filter-group">
              <span class="filter-label">Time range</span>
              <input type="number" v-model="filterFrom" :placeholder="String(dataRange.min)" />
              <span class="filter-to">to</span>
              <input type="number" v-model="filterTo" :placeholder="String(dataRange.max)" />
            </div>
            <button class="filter-reset" @click="resetFilters"><RotateCcw :size="12" /> Reset</button>
          </div>
          <div ref="timelineEl" class="timeline-card"><div class="timeline-title"><div><span class="live-dot"></span> {{ parsed.title }}</div><span class="scale-note">relative scale</span></div><div v-if="parsed.subtitle" class="timeline-subtitle">{{ parsed.subtitle }}</div><div class="timeline-scroll"><div class="timeline-canvas" :style="{ minWidth: canvasWidth }"><div class="axis"><span v-for="tick in ticks" :key="tick" class="tick" :style="{ left: position(tick) + '%' }">{{ tick > 0 ? '+' + tick : tick }}</span></div><div class="lanes-wrap"><div v-for="(lane, index) in visibleLanes" :key="lane.name + index" class="lane"><div class="lane-label" :style="{ marginLeft: position(laneStart(lane)) + '%' }"><span>{{ lane.name }}</span><em v-if="lane.events.length">starts {{ laneStart(lane) > 0 ? '+' + laneStart(lane) : laneStart(lane) }}</em></div><div class="lane-track" :style="{ backgroundColor: lane.color + '88', marginLeft: position(laneStart(lane)) + '%', width: (100 - position(laneStart(lane))) + '%' }"><div v-for="event in lane.events" :key="event.id" class="event" :class="eventEdge(event, lane)" :style="{ left: lanePosition(event.position, lane) + '%' }"><div class="event-label"><b>{{ event.id }}</b><strong>{{ eventTitle(event.label) }}</strong><small>{{ eventDetail(event.label) }}</small></div><div class="event-node" :style="{ backgroundColor: lane.color }"></div></div></div></div><div class="connector-overlay"><div v-for="c in connectors" :key="c.id" class="connector" :style="{ left: c.left + '%', top: c.top + 'px', height: c.height + 'px' }"><span></span></div></div></div></div></div><div v-if="!visibleLanes.some(l => l.events.length)" class="empty-state">{{ filtersActive ? 'No events match the current filters.' : 'Add events in Markdown to see them here.' }}</div></div>
        </div>
      </section>
    </main>
    <main v-else class="changelog-page">
      <section class="changelog-card">
        <h1>Changelog</h1>
        <p class="changelog-intro">Notable updates to Timeline.md.</p>
        <div v-for="entry in changelog" :key="entry.version" class="changelog-entry">
          <div class="changelog-entry-head"><span class="changelog-version">v{{ entry.version }}</span><span class="changelog-date">{{ entry.date }}</span></div>
          <ul><li v-for="change in entry.changes" :key="change">{{ change }}</li></ul>
        </div>
      </section>
    </main>
  </div>
</template>
