<script setup>
import { computed, nextTick, ref, watch } from 'vue'
import { toPng } from 'html-to-image'
import { Download, FileText, HelpCircle, RotateCcw, Sparkles, Upload } from 'lucide-vue-next'

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
const activeTab = ref('editor')
const notice = ref('')
const timelineEl = ref(null)

const parsed = computed(() => parseMarkdown(markdown.value))
const range = computed(() => {
  const values = parsed.value.lanes.flatMap(lane => lane.events.map(event => event.position))
  if (!values.length) return { min: -4, max: 16 }
  const min = Math.min(...values), max = Math.max(...values)
  return { min: Math.min(0, min), max: Math.max(10, max) }
})
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
function newTimeline() { markdown.value = '# New timeline\n\n## Act 1 — Start\n- [A1] First event @ 0'; fileName.value = 'untitled-timeline.md'; activeTab.value = 'editor' }
function loadFile(event) {
  const file = event.target.files?.[0]; if (!file) return
  const reader = new FileReader(); reader.onload = () => { markdown.value = reader.result; fileName.value = file.name; activeTab.value = 'editor' }; reader.readAsText(file)
}
function exportMarkdown() {
  const blob = new Blob([markdown.value], { type: 'text/markdown;charset=utf-8' })
  const link = document.createElement('a'); link.download = fileName.value || 'timeline.md'; link.href = URL.createObjectURL(blob); link.click(); URL.revokeObjectURL(link.href)
}
async function exportPng() {
  if (!timelineEl.value) return
  try { const dataUrl = await toPng(timelineEl.value, { pixelRatio: 2, backgroundColor: '#f8fafc' }); const link = document.createElement('a'); link.download = `${fileName.value.replace(/\.md$/i, '')}.png`; link.href = dataUrl; link.click(); notice.value = 'PNG exported'; setTimeout(() => notice.value = '', 1800) } catch { notice.value = 'Export failed' }
}
watch(markdown, () => { localStorage.setItem('timeline-md-draft', markdown.value) })
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand"><div class="brand-mark"><Sparkles :size="17" /></div><span>Timeline<span class="brand-accent">.md</span></span></div>
      <div class="header-actions"><span v-if="notice" class="notice">{{ notice }}</span><button class="icon-btn" title="Help"><HelpCircle :size="18" /></button><button class="avatar">M</button></div>
    </header>
    <main class="workspace">
      <aside class="sidebar">
        <div class="sidebar-title"><span>MY TIMELINES</span><button class="small-icon" @click="newTimeline"><Plus :size="16" /></button></div>
        <button class="timeline-file active"><FileText :size="16" /><span>{{ fileName }}</span><span v-if="!saved" class="unsaved">•</span></button>
        <div class="sidebar-footer"><p>Everything is saved in your browser.</p><button class="reset-btn" @click="reset"><RotateCcw :size="14" /> Reset example</button></div>
      </aside>
      <section class="content">
        <div class="page-heading"><div><p class="eyebrow">VISUAL PLANNING TOOL</p><h1>Build your timeline<span class="heading-dot">.</span></h1><p class="subheading">Write simple Markdown. See your story unfold.</p></div><div class="page-actions"><label class="btn secondary"><Upload :size="16" /> Import .md<input type="file" accept=".md,.markdown,text/markdown" @change="loadFile" hidden /></label><button class="btn primary" @click="exportPng"><Download :size="16" /> Export PNG</button></div></div>
        <div class="tabs"><button :class="{ selected: activeTab === 'editor' }" @click="activeTab = 'editor'"><FileText :size="15" /> Markdown</button><button :class="{ selected: activeTab === 'preview' }" @click="activeTab = 'preview'"><LayoutTemplate :size="15" /> Preview</button><span class="tab-hint">{{ parsed.lanes.reduce((n, l) => n + l.events.length, 0) }} events · positions relative to first event</span></div>
        <div v-if="activeTab === 'editor'" class="editor-card"><div class="editor-head"><div class="source-heading"><strong>Story source</strong><span class="language-pill">MARKDOWN</span><label class="inline-action">Import .md<input type="file" accept=".md,.markdown,text/markdown" @change="loadFile" hidden /></label><button class="inline-action" @click="exportMarkdown">Export .md</button></div></div><textarea v-model="markdown" spellcheck="false" aria-label="Markdown timeline source"></textarea><div class="syntax-help"><span>Tip</span> Use <code>[-300] A1: Event name &gt; detail</code> and <code>## Lane | #color</code>.</div></div>
        <div class="preview-heading"><div><h2>Live preview</h2><p>{{ parsed.lanes.reduce((n, l) => n + l.events.length, 0) }} events · positions relative to first event</p></div><div class="preview-actions"><div class="range-label">{{ range.min }} <span>→</span> {{ range.max }} <small>units</small></div><button class="preview-export" @click="exportPng"><Download :size="14" /> Export PNG</button></div></div>
        <div ref="timelineEl" class="timeline-card"><div class="timeline-title"><div><span class="live-dot"></span> {{ parsed.title }}</div><span class="scale-note">relative scale</span></div><div v-if="parsed.subtitle" class="timeline-subtitle">{{ parsed.subtitle }}</div><div class="timeline-scroll"><div class="timeline-canvas" :style="{ minWidth: Math.max(720, ticks.length * 86) + 'px' }"><div class="axis"><span v-for="tick in ticks" :key="tick" class="tick" :style="{ left: position(tick) + '%' }">{{ tick > 0 ? '+' + tick : tick }}</span></div><div v-for="(lane, index) in parsed.lanes" :key="lane.name + index" class="lane"><div class="lane-label" :style="{ marginLeft: position(laneStart(lane)) + '%' }"><span>{{ lane.name }}</span><em v-if="lane.events.length">starts {{ laneStart(lane) > 0 ? '+' + laneStart(lane) : laneStart(lane) }}</em></div><div class="lane-track" :style="{ backgroundColor: lane.color + '88', marginLeft: position(laneStart(lane)) + '%', width: (100 - position(laneStart(lane))) + '%' }"><div v-for="event in lane.events" :key="event.id" class="event" :class="eventEdge(event, lane)" :style="{ left: lanePosition(event.position, lane) + '%' }"><div class="event-label"><b>{{ event.id }}</b><strong>{{ eventTitle(event.label) }}</strong><small>{{ eventDetail(event.label) }}</small></div><div class="event-node" :style="{ backgroundColor: lane.color }"></div></div></div><div v-if="index < parsed.lanes.length - 1" class="connector-layer"><div v-for="link in parsed.links.filter(item => parsed.lanes[index].events.some(e => e.id === item.from))" :key="'c' + link.from" class="connector" :style="{ left: position(parsed.lanes[index].events.find(e => e.id === link.from)?.position || 0) + '%' }"><span></span></div></div></div></div></div><div v-if="!parsed.lanes.some(l => l.events.length)" class="empty-state">Add events in Markdown to see them here.</div></div>
        <div class="format-note"><span class="note-icon">i</span><div><strong>Timeline format</strong><p>Each event uses <code>- [ID] Label @ position</code>. Position is a number relative to your timeline — it can be negative or positive.</p></div></div>
      </section>
    </main>
  </div>
</template>
