<!--
  App.vue（HW2：動態視覺化）
  狀態集中在這裡：zoom（刷選範圍）、yearA / yearB（年度比較）、hover（同步提示）
  任何狀態一改，各元件透過 props 自動更新；放大與長條變化都用 D3 補間做成動畫
-->
<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import * as d3 from 'd3'
import { loadPopulation } from './utils/population.js'
import { useTween } from './composables/useTween.js'
import StatCards from './components/StatCards.vue'
import LineChart from './components/LineChart.vue'
import HoverTooltip from './components/HoverTooltip.vue'
import BrushOverview from './components/BrushOverview.vue'
import AgeCompareChart from './components/AgeCompareChart.vue'

const data = ref([])
const error = ref('')
onMounted(async () => {
  try { data.value = await loadPopulation() }
  catch (e) { error.value = '資料載入失敗：' + e.message }
})

const fmtN = d3.format(',')
const fmtWan = d => d === 0 ? '0' : (d / 10000).toFixed(d < 100000 ? 1 : 0) + ' 萬'

/* ---------- 從資料算出的重點 ---------- */
const fullDomain = computed(() => data.value.length ? d3.extent(data.value, d => d.year) : [1958, 2025])
const last = computed(() => data.value.at(-1))
const peak = computed(() => d3.greatest(data.value, d => d.births))
const firstAge = computed(() => data.value.find(d => d.pct_65p != null))

/* ---------- 1. 刷選放大 ---------- */
const zoom = ref(null)                                        // null = 顯示全部；否則 [起, 迄]
const targetDomain = computed(() => zoom.value ?? fullDomain.value)
const viewDomain = useTween(targetDomain, { duration: 700 })  // x 範圍用 D3 補間，主圖平滑放大縮小
const zoomLabel = computed(() => zoom.value ? `${zoom.value[0]}–${zoom.value[1]} 年` : `全部（${fullDomain.value[0]}–${fullDomain.value[1]} 年）`)

/* ---------- 2. 年度比較 ---------- */
const yearA = ref(1974)
const yearB = ref(2025)
watch(data, () => { if (firstAge.value) { yearA.value = firstAge.value.year; yearB.value = last.value.year } })
const ageYears = computed(() => firstAge.value ? [firstAge.value.year, last.value.year] : [1974, 2025])
const clampAge = yr => Math.max(ageYears.value[0], Math.min(ageYears.value[1], yr))
function pickYear(yr) { yearB.value = clampAge(yr) }          // 點主圖 → 設為比較年份 B
const markYears = computed(() => [
  { year: yearA.value, label: `A ${yearA.value}`, cls: 'a' },
  { year: yearB.value, label: `B ${yearB.value}`, cls: 'b' },
])

/* ---------- 文字與數字卡 ---------- */
const lede = computed(() => {
  if (!last.value) return ''
  const L = last.value, P = peak.value, F = firstAge.value
  return `${P.year} 年台灣一年出生 ${fmtN(P.births)} 人；到了 ${L.year} 年只剩 ${fmtN(L.births)} 人。`
    + `同一段時間，65 歲以上人口比例從 ${F.year} 年的 ${F.pct_65p.toFixed(1)}% 爬升到 ${L.pct_65p.toFixed(1)}%。`
    + `拖曳下方總覽圖可以放大任一段時期，點擊圖表或調整滑桿可以比較任兩年的人口結構。`
})
const stats = computed(() => {
  if (!last.value) return []
  const L = last.value, P = peak.value, F = firstAge.value
  return [
    { k: `${L.year} 年出生數`, v: fmtN(L.births), d: `${P.year} 年高峰的 ${(L.births / P.births * 100).toFixed(0)}%` },
    { k: `${L.year} 年 65 歲以上比例`, v: L.pct_65p.toFixed(1) + '%', d: `約每 ${Math.round(100 / L.pct_65p)} 人就有 1 位長者` },
    { k: `${L.year} 年 0–11 歲比例`, v: L.pct_0_11.toFixed(1) + '%', d: `${F.year} 年為 ${F.pct_0_11.toFixed(1)}%` },
  ]
})

/* ---------- 標註 ---------- */
const birthNotes = computed(() => {
  if (!last.value) return []
  return [
    { year: peak.value.year, title: `${peak.value.year} 年高峰`, sub: fmtN(peak.value.births) + ' 人', anchor: 'middle', dy: -30 },
    { year: 2012, title: '2012 龍年', sub: fmtN(data.value.find(d => d.year === 2012)?.births ?? 0) + ' 人', anchor: 'middle', dy: -30 },
    { year: last.value.year, title: `${last.value.year} 年新低`, sub: fmtN(last.value.births) + ' 人', anchor: 'start', dx: 10, dy: -4 },
  ]
})
const THRESHOLDS = [{ v: 7, name: '高齡化社會' }, { v: 14, name: '高齡社會' }, { v: 20, name: '超高齡社會' }]
const elderNotes = computed(() => THRESHOLDS
  .map((t, i) => ({ t, i, d: data.value.find(d => d.pct_65p != null && d.pct_65p >= t.v) }))
  .filter(o => o.d)
  .map(({ t, i, d }) => i === 2
    ? { year: d.year, title: `${d.year} 年跨過 ${t.v}%`, anchor: 'start', dx: 10, dy: 4 }
    : { year: d.year, title: `${d.year} 年跨過 ${t.v}%`, anchor: 'end', dx: -8, dy: -8 }))

/* ---------- Hover ---------- */
const hover = ref(null)
const hoverDatum = computed(() => hover.value && data.value.find(d => d.year === hover.value.year))
</script>

<template>
  <main>
    <h1>出生越來越少，老人越來越多：<br>互動探索台灣的人口結構變化</h1>
    <p class="lede">{{ lede }}</p>
    <p v-if="error" class="error">{{ error }}</p>

    <template v-if="data.length">
      <StatCards :items="stats" />

      <!-- ===== 時間序列 + 刷選放大 ===== -->
      <section class="card">
        <div class="card-head">
          <div>
            <h2><span class="sw" style="background: var(--births)"></span>每年出生數</h2>
            <p class="sub">單位：人（按登記日期）・目前顯示：{{ zoomLabel }}</p>
          </div>
          <button v-if="zoom" class="btn" @click="zoom = null">顯示全部</button>
        </div>
        <LineChart :rows="data" value-key="births" color="births" :x-domain="viewDomain"
                   :y-format="fmtWan" :annotations="birthNotes" :mark-years="markYears" label="台灣每年出生數折線圖"
                   :hover-year="hover?.year ?? null" @hover="hover = $event" @leave="hover = null" @pick="pickYear" />

        <h2 class="second"><span class="sw" style="background: var(--old)"></span>65 歲以上人口比例</h2>
        <p class="sub">單位：%；虛線為聯合國定義的高齡化門檻</p>
        <LineChart :rows="data" value-key="pct_65p" color="old" :x-domain="viewDomain" :y-domain="[0, 25]"
                   :y-format="d => d + '%'" :thresholds="THRESHOLDS" :annotations="elderNotes" :mark-years="markYears"
                   :gap-note="`${firstAge?.year} 年前無資料`" label="台灣65歲以上人口比例折線圖"
                   :hover-year="hover?.year ?? null" @hover="hover = $event" @leave="hover = null" @pick="pickYear" />

        <div class="overview">
          <p class="sub"><b>總覽：</b>在下圖拖曳選取年份區間即可放大上方兩張圖；點選框外空白處取消</p>
          <BrushOverview :rows="data" :full-domain="fullDomain" v-model="zoom" />
        </div>
      </section>

      <!-- ===== 年度比較 ===== -->
      <section class="card">
        <h2>兩個年份的人口結構比較</h2>
        <p class="sub">四個年齡層加起來為 100%；拖曳滑桿，或直接點擊上方折線圖設定年份 B</p>
        <div class="controls">
          <label class="ctl">
            <span class="chip a"></span>年份 A：<b>{{ yearA }}</b>
            <input type="range" :min="ageYears[0]" :max="ageYears[1]" v-model.number="yearA" />
          </label>
          <label class="ctl">
            <span class="chip b"></span>年份 B：<b>{{ yearB }}</b>
            <input type="range" :min="ageYears[0]" :max="ageYears[1]" v-model.number="yearB" />
          </label>
        </div>
        <AgeCompareChart :rows="data" :year-a="yearA" :year-b="yearB" />
      </section>
    </template>

    <p class="source">資料來源：內政部戶政司「出生數及粗出生率（按登記日期，全國）」、「人口年齡結構重要指標」。
      出生數 1958–2025 年，人口年齡結構 1974–2025 年；原始 XLS 以 Excel 另存 CSV，清理與合併以 D3.js 完成，介面以 Vue 3 元件化。</p>

    <HoverTooltip :datum="hoverDatum" :client-x="hover?.clientX ?? 0" :client-y="hover?.clientY ?? 0" />
  </main>
</template>

<style scoped>
main { max-width: 960px; margin: 0 auto; padding: 32px 16px 48px; }
h1 { font-size: 28px; line-height: 1.35; margin: 0 0 8px; }
.lede { color: var(--text-2); font-size: 15px; line-height: 1.7; margin: 0 0 20px; min-height: 1.7em; }
.error { color: #d03b3b; }
.card { background: var(--panel); border: 1px solid var(--border); border-radius: 12px; padding: 16px 18px; margin-bottom: 20px; }
.card h2 { font-size: 15px; margin: 0 0 2px; display: flex; align-items: center; gap: 8px; }
.card h2.second { margin-top: 14px; }
.card-head { display: flex; justify-content: space-between; align-items: flex-start; gap: 12px; }
.sw { width: 12px; height: 3px; border-radius: 2px; display: inline-block; }
.sub { font-size: 12px; color: var(--text-2); margin: 0 0 6px; }
.overview { margin-top: 10px; padding-top: 10px; border-top: 1px dashed var(--border); }
.btn { font: inherit; font-size: 12px; padding: 5px 12px; border-radius: 999px; cursor: pointer;
  background: var(--surface); color: var(--text-1); border: 1px solid var(--border); }
.btn:hover { border-color: var(--text-3); }
.controls { display: flex; flex-wrap: wrap; gap: 12px 32px; margin: 8px 0 4px; }
.ctl { display: flex; align-items: center; gap: 8px; font-size: 13px; color: var(--text-2); }
.ctl b { color: var(--text-1); font-variant-numeric: tabular-nums; min-width: 3em; display: inline-block; }
.ctl input { width: 220px; accent-color: var(--text-2); }
.chip { width: 12px; height: 12px; border-radius: 3px; display: inline-block; }
.chip.a { background: var(--cmp-a); }
.chip.b { background: var(--cmp-b); }
.source { font-size: 12px; color: var(--text-3); margin-top: 4px; line-height: 1.6; }
</style>
