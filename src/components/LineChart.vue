<!--
  通用折線＋面積圖元件（HW2 版）
  分工：D3 負責「算」（比例尺、路徑字串、刻度、找最近年份），Vue 負責「畫」（template 產生 SVG）
  HW2 新增：x 範圍可由父元件動態改變（刷選放大）、clipPath 裁切、點擊選年份
-->
<script setup>
import { computed, ref } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  rows:        { type: Array, required: true },
  valueKey:    { type: String, required: true },
  color:       { type: String, required: true },   // CSS 變數前綴：'births' 或 'old'
  xDomain:     { type: Array, required: true },    // 目前顯示的年份範圍（會被刷選改變，且已做補間動畫）
  yDomain:     { type: Array, default: null },
  yFormat:     { type: Function, default: d => d },
  thresholds:  { type: Array, default: () => [] },
  annotations: { type: Array, default: () => [] },
  gapNote:     { type: String, default: '' },
  hoverYear:   { type: Number, default: null },
  markYears:   { type: Array, default: () => [] },  // 年度比較選中的年份 [{ year, label, cls }]
  label:       { type: String, default: '' },
})
const emit = defineEmits(['hover', 'leave', 'pick'])

const W = 900, H = 290
const M = { t: 24, r: 110, b: 28, l: 56 }
const clipId = `clip-${Math.random().toString(36).slice(2, 8)}`   // 每個元件實例各自的 clipPath

/* ---------- D3：資料與比例尺 ---------- */
const value = d => d[props.valueKey]
const series = computed(() => props.rows.filter(d => value(d) != null))

const x = computed(() => d3.scaleLinear(props.xDomain, [M.l, W - M.r]))
const y = computed(() => {
  const domain = props.yDomain ?? [0, d3.max(series.value, value)]   // y 用全部資料，放大時不變，比較才公平
  const s = d3.scaleLinear(domain, [H - M.b, M.t])
  return props.yDomain ? s : s.nice()
})

const inView = yr => yr >= props.xDomain[0] - 1e-6 && yr <= props.xDomain[1] + 1e-6
const visible = computed(() => series.value.filter(d => inView(d.year)))

const xTicks = computed(() => x.value.ticks(Math.min(12, Math.ceil(props.xDomain[1] - props.xDomain[0]))).filter(Number.isInteger))
const yTicks = computed(() => y.value.ticks(5))

const linePath = computed(() => d3.line().x(d => x.value(d.year)).y(d => y.value(value(d)))(series.value))
const areaPath = computed(() => d3.area().x(d => x.value(d.year)).y0(y.value(0)).y1(d => y.value(value(d)))(series.value))

const notes = computed(() => props.annotations
  .map(a => ({ ...a, d: series.value.find(d => d.year === a.year) }))
  .filter(a => a.d && inView(a.year))
  .map(a => ({ ...a, cx: x.value(a.year), cy: y.value(value(a.d)) })))

const gap = computed(() => {
  const first = series.value[0]
  if (!props.gapNote || !first || first.year <= props.xDomain[0]) return null
  const x0 = x.value(props.xDomain[0]), x1 = Math.min(x.value(first.year), W - M.r)
  if (x1 - x0 < 80) return null
  return { x: (x0 + x1) / 2, y: y.value(y.value.domain()[1] * 0.12) }
})

const marks = computed(() => props.markYears.filter(m => inView(m.year)).map(m => ({ ...m, cx: x.value(m.year) })))

/* ---------- Hover / 點擊 ---------- */
const svgRef = ref(null)
const bisect = d3.bisector(d => d.year).center

function nearest(event) {
  if (!visible.value.length) return null
  const [mx] = d3.pointer(event, svgRef.value)
  return visible.value[bisect(visible.value, x.value.invert(mx))]
}
function onMove(event) {
  const d = nearest(event)
  if (d) emit('hover', { year: d.year, clientX: event.clientX, clientY: event.clientY })
}
function onClick(event) {
  const d = nearest(event)
  if (d) emit('pick', d.year)
}

const hoverPoint = computed(() => {
  if (props.hoverYear == null || !inView(props.hoverYear)) return null
  const d = series.value.find(r => r.year === props.hoverYear)
  return d ? { cx: x.value(d.year), cy: y.value(value(d)) } : { cx: x.value(props.hoverYear), cy: null }
})
</script>

<template>
  <svg ref="svgRef" :viewBox="`0 0 ${W} ${H}`" role="img" :aria-label="label">
    <defs>
      <clipPath :id="clipId"><rect :x="M.l" :y="0" :width="W - M.l - M.r" :height="H" /></clipPath>
    </defs>

    <g class="axis">
      <g v-for="t in yTicks" :key="'y' + t">
        <line :x1="M.l" :x2="W - M.r" :y1="y(t)" :y2="y(t)" class="grid" />
        <text :x="M.l - 8" :y="y(t)" dy="0.32em" text-anchor="end">{{ yFormat(t) }}</text>
      </g>
      <text v-for="t in xTicks" :key="'x' + t" :x="x(t)" :y="H - M.b + 18" text-anchor="middle">{{ t }}</text>
    </g>

    <g v-for="t in thresholds" :key="'th' + t.v">
      <line :x1="M.l" :x2="W - M.r" :y1="y(t.v)" :y2="y(t.v)" class="threshold" />
      <text :x="M.l + 6" :y="y(t.v) + 14" class="th-label">{{ t.v }}%　{{ t.name }}</text>
    </g>

    <!-- 線與面積用 clipPath 裁切，放大時不會畫出框外 -->
    <g :clip-path="`url(#${clipId})`">
      <path :d="areaPath" :style="{ fill: `var(--${color}-fill)` }" opacity="0.6" />
      <path :d="linePath" :style="{ stroke: `var(--${color})` }" fill="none" stroke-width="2" />
      <!-- 年度比較選中的年份 -->
      <g v-for="m in marks" :key="'m' + m.label">
        <line :x1="m.cx" :x2="m.cx" :y1="M.t" :y2="H - M.b" :class="['mark', m.cls]" />
      </g>
    </g>
    <text v-for="m in marks" :key="'ml' + m.label" :x="m.cx" :y="M.t - 8" text-anchor="middle" :class="['mark-label', m.cls]">{{ m.label }}</text>

    <text v-if="gap" :x="gap.x" :y="gap.y" text-anchor="middle" class="anno-sub">{{ gapNote }}</text>

    <g v-for="n in notes" :key="'n' + n.year">
      <circle :cx="n.cx" :cy="n.cy" r="4.5" :style="{ fill: `var(--${color})` }" class="ring" />
      <text :x="n.cx + (n.dx ?? 0)" :y="n.cy + (n.dy ?? 0)" :text-anchor="n.anchor ?? 'start'">
        <tspan class="anno">{{ n.title }}</tspan>
        <tspan v-if="n.sub" class="anno-sub" :x="n.cx + (n.dx ?? 0)" dy="15">{{ n.sub }}</tspan>
      </text>
    </g>

    <g v-if="hoverPoint" pointer-events="none">
      <line :x1="hoverPoint.cx" :x2="hoverPoint.cx" :y1="M.t" :y2="H - M.b" class="hover-line" />
      <circle v-if="hoverPoint.cy != null" :cx="hoverPoint.cx" :cy="hoverPoint.cy" r="5"
              :style="{ fill: `var(--${color})` }" class="ring" />
    </g>

    <rect :x="M.l" :y="M.t" :width="W - M.l - M.r" :height="H - M.t - M.b"
          fill="transparent" style="cursor: pointer"
          @pointermove="onMove" @pointerleave="emit('leave')" @click="onClick" />
  </svg>
</template>

<style scoped>
svg { display: block; width: 100%; height: auto; overflow: visible; }
.axis text { fill: var(--text-2); font-size: 11px; }
.grid { stroke: var(--grid); }
.threshold { stroke: var(--threshold); stroke-dasharray: 4 4; }
.th-label { fill: var(--text-2); font-size: 11px; paint-order: stroke; stroke: var(--panel); stroke-width: 4px; }
.anno { fill: var(--text-1); font-size: 12px; font-weight: 600; }
.anno-sub { fill: var(--text-2); font-size: 11px; }
.ring { stroke: var(--panel); stroke-width: 2; }
.hover-line { stroke: var(--text-3); stroke-dasharray: 3 3; }
.mark { stroke-width: 2; opacity: 0.85; }
.mark.a { stroke: var(--cmp-a); }
.mark.b { stroke: var(--cmp-b); }
.mark-label { font-size: 11px; font-weight: 700; fill: var(--text-1); }
</style>
