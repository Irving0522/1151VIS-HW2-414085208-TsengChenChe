<!--
  年度比較長條圖：兩個年份 × 四個年齡層（0–11、12–17、18–64、65 歲以上，四組加起來 = 100%）
  切換年份時，長條高度與數字用 useTween（d3.interpolate + d3.timer）平滑變化
-->
<script setup>
import { computed } from 'vue'
import * as d3 from 'd3'
import { useTween } from '../composables/useTween.js'

const props = defineProps({
  rows:  { type: Array, required: true },
  yearA: { type: Number, required: true },
  yearB: { type: Number, required: true },
})

const GROUPS = [
  { key: 'g0_11',  label: '0–11 歲' },
  { key: 'g12_17', label: '12–17 歲' },
  { key: 'g18_64', label: '18–64 歲' },
  { key: 'g65p',   label: '65 歲以上' },
]

/* 把一年的資料換成四個互不重疊的年齡層比例；18–64 歲 = 100 − 其他三組 */
function shares(year) {
  const d = props.rows.find(r => r.year === year)
  if (!d || d.pct_65p == null) return [0, 0, 0, 0]
  return [d.pct_0_11, d.pct_12_17, 100 - d.pct_0_11 - d.pct_12_17 - d.pct_65p, d.pct_65p]
}

// 目標值一變，畫面上的值就用 D3 補間動畫追過去
const valsA = useTween(() => shares(props.yearA))
const valsB = useTween(() => shares(props.yearB))
const targetA = computed(() => shares(props.yearA))
const targetB = computed(() => shares(props.yearB))

const W = 900, H = 330
const M = { t: 30, r: 20, b: 64, l: 56 }

const x0 = d3.scaleBand(GROUPS.map(g => g.key), [M.l, W - M.r]).paddingInner(0.28).paddingOuter(0.1)
const x1 = d3.scaleBand(['a', 'b'], [0, x0.bandwidth()]).padding(0.08)
const y = d3.scaleLinear([0, 80], [H - M.b, M.t])
const yTicks = y.ticks(4)

const bars = computed(() => GROUPS.map((g, i) => ({
  ...g,
  x: x0(g.key),
  a: valsA.value[i], b: valsB.value[i],
  delta: targetB.value[i] - targetA.value[i],
})))

const fmtDelta = v => 'B − A：' + (v >= 0 ? '+' : '−') + Math.abs(v).toFixed(1) + ' 個百分點'
</script>

<template>
  <svg :viewBox="`0 0 ${W} ${H}`" role="img" aria-label="兩個年份各年齡層人口比例比較">
    <g class="axis">
      <g v-for="t in yTicks" :key="t">
        <line :x1="M.l" :x2="W - M.r" :y1="y(t)" :y2="y(t)" class="grid" />
        <text :x="M.l - 8" :y="y(t)" dy="0.32em" text-anchor="end">{{ t }}%</text>
      </g>
    </g>

    <g v-for="g in bars" :key="g.key" :transform="`translate(${g.x},0)`">
      <!-- 年份 A -->
      <rect :x="x1('a')" :y="y(g.a)" :width="x1.bandwidth()" :height="y(0) - y(g.a)" rx="3" class="bar a">
        <title>{{ yearA }} 年 {{ g.label }}：{{ g.a.toFixed(2) }}%</title>
      </rect>
      <text :x="x1('a') + x1.bandwidth() / 2" :y="y(g.a) - 6" text-anchor="middle" class="val">{{ g.a.toFixed(1) }}%</text>
      <!-- 年份 B -->
      <rect :x="x1('b')" :y="y(g.b)" :width="x1.bandwidth()" :height="y(0) - y(g.b)" rx="3" class="bar b">
        <title>{{ yearB }} 年 {{ g.label }}：{{ g.b.toFixed(2) }}%</title>
      </rect>
      <text :x="x1('b') + x1.bandwidth() / 2" :y="y(g.b) - 6" text-anchor="middle" class="val">{{ g.b.toFixed(1) }}%</text>

      <text :x="x0.bandwidth() / 2" :y="H - M.b + 20" text-anchor="middle" class="group">{{ g.label }}</text>
      <text :x="x0.bandwidth() / 2" :y="H - M.b + 38" text-anchor="middle" class="delta">{{ fmtDelta(g.delta) }}</text>
    </g>
    <line :x1="M.l" :x2="W - M.r" :y1="y(0)" :y2="y(0)" class="baseline" />
  </svg>
</template>

<style scoped>
svg { display: block; width: 100%; height: auto; }
.axis text { fill: var(--text-2); font-size: 11px; }
.grid { stroke: var(--grid); }
.baseline { stroke: var(--text-3); }
.bar.a { fill: var(--cmp-a); }
.bar.b { fill: var(--cmp-b); }
.val { fill: var(--text-1); font-size: 12px; font-weight: 600; font-variant-numeric: tabular-nums; }
.group { fill: var(--text-1); font-size: 13px; font-weight: 600; }
.delta { fill: var(--text-2); font-size: 11px; }
</style>
