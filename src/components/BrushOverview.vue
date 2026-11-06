<!--
  總覽小圖＋刷選（brush）
  在這張小圖上拖曳出年份區間 → 上方兩張主圖放大到該區間
  刷選互動交給 d3.brushX（D3 直接管理這個 <g>），其餘仍由 Vue template 繪製
-->
<script setup>
import { computed, ref, onMounted, watch } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  rows:       { type: Array, required: true },
  fullDomain: { type: Array, required: true },       // 全部年份範圍
  modelValue: { type: Array, default: null },        // 目前刷選範圍 [起, 迄]，null 代表全部
})
const emit = defineEmits(['update:modelValue'])

const W = 900, H = 78
const M = { t: 6, r: 110, b: 22, l: 56 }

const x = computed(() => d3.scaleLinear(props.fullDomain, [M.l, W - M.r]))
const yB = computed(() => d3.scaleLinear([0, d3.max(props.rows, d => d.births)], [H - M.b, M.t]))
const yO = computed(() => d3.scaleLinear([0, 25], [H - M.b, M.t]))
const birthsPath = computed(() => d3.area().x(d => x.value(d.year)).y0(H - M.b).y1(d => yB.value(d.births))(props.rows))
const elderPath = computed(() => d3.line().x(d => x.value(d.year)).y(d => yO.value(d.pct_65p))
  (props.rows.filter(d => d.pct_65p != null)))
const ticks = computed(() => x.value.ticks(12))

/* ---------- d3.brushX ---------- */
const brushG = ref(null)
let brush, internal = false

onMounted(() => {
  brush = d3.brushX()
    .extent([[M.l, M.t], [W - M.r, H - M.b]])
    .on('end', event => {
      if (internal || !event.sourceEvent) return            // 程式自己移動 brush 時不回傳
      if (!event.selection) { emit('update:modelValue', null); return }
      // 把像素換回年份，並對齊到整數年
      let [y0, y1] = event.selection.map(px => Math.round(x.value.invert(px)))
      if (y1 - y0 < 3) y1 = Math.min(props.fullDomain[1], y0 + 3)   // 至少 3 年，避免太窄
      emit('update:modelValue', [y0, y1])
      moveBrush([y0, y1])                                    // 吸附到整數年的位置
    })
  d3.select(brushG.value).call(brush)
  moveBrush(props.modelValue)
})

function moveBrush(range) {
  internal = true
  d3.select(brushG.value).transition().duration(300)
    .call(brush.move, range ? range.map(x.value) : null)
    .on('end interrupt', () => { internal = false })
}

// 外部（例如「顯示全部」按鈕）改了範圍時，同步移動 brush
watch(() => props.modelValue, v => { if (!internal) moveBrush(v) })
</script>

<template>
  <svg :viewBox="`0 0 ${W} ${H}`" role="img" aria-label="年份總覽與刷選">
    <path :d="birthsPath" class="b-area" />
    <path :d="elderPath" class="o-line" />
    <text v-for="t in ticks" :key="t" :x="x(t)" :y="H - 6" text-anchor="middle" class="tick">{{ t }}</text>
    <g ref="brushG" class="brush" />
  </svg>
</template>

<style scoped>
svg { display: block; width: 100%; height: auto; }
.b-area { fill: var(--births-fill); stroke: var(--births); stroke-width: 1; }
.o-line { fill: none; stroke: var(--old); stroke-width: 1.5; }
.tick { fill: var(--text-2); font-size: 10px; }
.brush :deep(.selection) { fill: var(--text-1); fill-opacity: 0.08; stroke: var(--text-2); stroke-width: 1; }
.brush :deep(.handle) { fill: var(--text-2); fill-opacity: 0.5; rx: 2; }
</style>
