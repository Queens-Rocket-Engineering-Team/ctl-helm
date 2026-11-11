<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref, shallowRef, watch } from 'vue'
import uPlot from 'uplot'
import 'uplot/dist/uPlot.min.css'

const props = defineProps({
  data: {
    type: Array,
    required: true,
  },
  color: {
    type: String,
    required: true,
  },
  fill: {
    type: String,
    default: 'transparent',
  },
  windowSec: {
    type: Number,
    default: 30,
  },
    showFit: {
    type: Boolean,
    default: true,
  },
})

const root = ref(null)
const plot = shallowRef(null)
let resizeObserver = null

const r2 = ref(null)

function fitLine(x, y) {
  let n = 0, sx = 0, sy = 0, sxx = 0, sxy = 0, syy = 0
  for (let i = 0; i < x.length; i++) {
    if (!Number.isFinite(x[i]) || !Number.isFinite(y[i])) continue
    n++
    sx += x[i]
    sy += y[i]
    sxx += x[i] * x[i]
    sxy += x[i] * y[i]
    syy += y[i] * y[i]
  }
  const d = n * sxx - sx * sx
  if (n < 2 || d === 0) return null
  const slope = (n * sxy - sx * sy) / d
  const varY = n * syy - sy * sy
  const r = varY > 0 ? (n * sxy - sx * sy) / Math.sqrt(d * varY) : 0
  return { slope, intercept: (sy - slope * sx) / n, r2: r * r }
}

function normalizedData() {
  const x = Array.isArray(props.data?.[0]) ? props.data[0] : []
  const y = Array.isArray(props.data?.[1]) ? props.data[1] : []
  if (!props.showFit) {
    r2.value = null
    return [x, y]
  }
  const f = fitLine(x, y)
  r2.value = f ? f.r2 : null
  const fit = f ? x.map((v) => f.slope * v + f.intercept) : x.map(() => null)
  return [x, y, fit]
}

function size() {
  const el = root.value
  return {
    width: Math.max(1, el?.clientWidth ?? 1),
    height: Math.max(1, el?.clientHeight ?? 1),
  }
}

function yRange(_u, min, max) {
  if (!Number.isFinite(min) || !Number.isFinite(max)) return [0, 1]
  if (min === max) {
    const pad = Math.max(Math.abs(min) * 0.05, 1)
    return [min - pad, max + pad]
  }
  const pad = (max - min) * 0.08
  return [min - pad, max + pad]
}

function yScale(data = normalizedData()) {
  const values = data[1].filter(Number.isFinite)
  if (values.length === 0) return { min: 0, max: 1 }

  let min = values[0]
  let max = values[0]
  for (const value of values) {
    min = Math.min(min, value)
    max = Math.max(max, value)
  }

  const [rangeMin, rangeMax] = yRange(null, min, max)
  return { min: rangeMin, max: rangeMax }
}

function xScale() {
  return { min: -props.windowSec, max: 0 }
}

function formatTick(value) {
  const abs = Math.abs(value)
  if (abs >= 1000) return value.toFixed(0)
  if (abs >= 10) return value.toFixed(1)
  return value.toFixed(2)
}

function options() {
  const { width, height } = size()
  const grid = 'rgba(128,128,128,0.12)'
  const ticks = 'rgba(128,128,128,0.22)'
  const text = 'rgba(128,128,128,0.8)'

  return {
    width,
    height,
    padding: [4, 14, 3, 0],
    legend: { show: false },
    cursor: {
      show: false,
      drag: { x: false, y: false },
    },
    scales: {
      x: { time: false, ...xScale() },
      y: { auto: true, range: yRange },
    },
    axes: [
      {
        stroke: text,
        grid: { stroke: grid, width: 1 },
        ticks: { stroke: ticks, width: 1 },
        size: 24,
        font: '8px Inter, system-ui, sans-serif',
        splits: () => [-props.windowSec, -Math.round(props.windowSec * 2 / 3), -Math.round(props.windowSec / 3), 0],
        values: (_u, vals) => vals.map((v) => `${v.toFixed(0)}s`),
      },
      {
        stroke: text,
        grid: { stroke: grid, width: 1 },
        ticks: { stroke: ticks, width: 1 },
        size: 34,
        font: '8px Inter, system-ui, sans-serif',
        values: (_u, vals) => vals.map(formatTick),
      },
    ],
    series: [
      {},
      {
        stroke: props.color,
        fill: props.fill,
        width: 1.5,
        points: { show: false },
        spanGaps: true,
      },
      ...(props.showFit
        ? [{
            stroke: 'red',
            width: 1.5,
            dash: [6, 4],
            points: { show: false },
            spanGaps: true,
          }]
        : []),
    
    ],
  }
}

function createPlot() {
  if (!root.value) return
  plot.value = new uPlot(options(), normalizedData(), root.value)
  plot.value.setScale('x', xScale())
  plot.value.setScale('y', yScale())
}

function resizePlot() {
  if (!plot.value) return
  plot.value.setSize(size())
}

onMounted(async () => {
  await nextTick()
  createPlot()

  if (root.value) {
    resizeObserver = new ResizeObserver(resizePlot)
    resizeObserver.observe(root.value)
  }
})

watch(
  () => props.data,
  () => {
    if (!plot.value) return
    const data = normalizedData()
    plot.value.setData(data, false)
    plot.value.setScale('x', xScale())
    plot.value.setScale('y', yScale(data))
  },
)

watch(
  () => [props.color, props.fill, props.showFit],
  () => {
    if (!plot.value) return
    plot.value.destroy()
    createPlot()
  },
)

watch(
  () => props.windowSec,
  () => {
    if (!plot.value) return
    plot.value.destroy()
    createPlot()
  },
)

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
  plot.value?.destroy()
})
</script>

<template>
  <div class="uplot-wrap">
    <div ref="root" class="uplot-chart" />
    <span v-if="showFit && r2 !== null" class="fit-r2">R² {{ r2.toFixed(3) }}</span>
  </div>
</template>

<style scoped>
.uplot-wrap {
  position: relative;
  width: 100%;
  height: 100%;
}

.uplot-chart {
  width: 100%;
  height: 100%;
}

.fit-r2 {
  position: absolute;
  top: 4px;
  right: 18px;
  z-index: 2;
  font-size: 10px;
  color: rgb(0, 38, 255)e;
  pointer-events: none;
}

.uplot-chart :deep(.uplot) {
  width: 100%;
  height: 100%;
}
</style>
