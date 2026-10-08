<script setup lang="ts">
import ECharts from 'vue-echarts';

import type { EChartsOption } from 'echarts';

import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { LineChart, ScatterChart } from 'echarts/charts';
import {
  DatasetComponent,
  GridComponent,
  LegendComponent,
  TooltipComponent,
} from 'echarts/components';

use([
  LineChart,
  ScatterChart,
  CanvasRenderer,
  DatasetComponent,
  GridComponent,
  LegendComponent,
  TooltipComponent,
]);

// Written by the self-improvement loop, one JSON object per line.
// Served as a static file next to index.html.
const DATA_URL = `${import.meta.env.BASE_URL}self_improve.jsonl`;

interface CycleMetrics {
  profit_pct: number;
  trades: number;
  max_drawdown_pct: number;
}

interface Cycle {
  ts: string;
  strategy: string;
  train_range: string;
  valid_range: string;
  baseline: CycleMetrics;
  candidate: CycleMetrics;
  lookahead_ok: boolean;
  promoted: boolean;
  note?: string;
  // 'params' cycles swap hyperopt parameters, 'code' cycles test a strategy variant
  kind?: 'params' | 'code';
  variant?: string;
  hypothesis?: string;
}

const CHART_BASELINE = 'Current params';
const CHART_CANDIDATE = 'Candidate';
const CHART_PROMOTED = 'Promoted';

const settingsStore = useSettingsStore();
const colorStore = useColorStore();

const cycles = ref<Cycle[]>([]);
const loading = ref(false);
const selectedStrategy = ref<string>();

async function load() {
  loading.value = true;
  try {
    const text = await (await fetch(DATA_URL, { cache: 'no-store' })).text();
    // A missing file falls back to index.html, which yields no parsable lines.
    cycles.value = text.split('\n').flatMap((line) => {
      try {
        const cycle = JSON.parse(line) as Cycle;
        return cycle?.strategy && cycle.baseline && cycle.candidate ? [cycle] : [];
      } catch {
        return [];
      }
    });
  } catch {
    cycles.value = [];
  } finally {
    loading.value = false;
  }
  const names = strategies.value;
  if (!selectedStrategy.value || !names.includes(selectedStrategy.value)) {
    selectedStrategy.value = names[0];
  }
}

const strategies = computed(() => [...new Set(cycles.value.map((c) => c.strategy))]);

const strategyCycles = computed(() =>
  cycles.value
    .filter((c) => c.strategy === selectedStrategy.value)
    .sort((a, b) => a.ts.localeCompare(b.ts)),
);

const summary = computed(() => {
  const all = strategyCycles.value;
  const promoted = all.filter((c) => c.promoted);
  return {
    total: all.length,
    promoted: promoted.length,
    last: all.at(-1),
    lastPromoted: promoted.at(-1),
  };
});

const chartOptions = computed((): EChartsOption => {
  const all = strategyCycles.value;
  return {
    backgroundColor: 'rgba(0, 0, 0, 0)',
    tooltip: { trigger: 'axis' },
    legend: { data: [CHART_BASELINE, CHART_CANDIDATE, CHART_PROMOTED], top: 0 },
    grid: { left: 60, right: 30, bottom: 40 },
    xAxis: { type: 'category', data: all.map((c) => formatTs(c.ts)) },
    yAxis: { type: 'value', name: 'Validation profit %', splitLine: { show: false } },
    series: [
      {
        type: 'line',
        name: CHART_BASELINE,
        animation: false,
        data: all.map((c) => c.baseline.profit_pct),
        itemStyle: { color: 'rgba(150,150,150,0.9)' },
      },
      {
        type: 'line',
        name: CHART_CANDIDATE,
        animation: false,
        data: all.map((c) => c.candidate.profit_pct),
      },
      {
        type: 'scatter',
        name: CHART_PROMOTED,
        animation: false,
        symbolSize: 14,
        data: all.map((c) => (c.promoted ? c.candidate.profit_pct : null)),
        itemStyle: { color: colorStore.colorProfit },
      },
    ],
  };
});

const tableColumns = [
  { accessorKey: 'ts', header: 'Cycle' },
  { accessorKey: 'kind', header: 'Change' },
  { accessorKey: 'train_range', header: 'Train range', meta: { class: { td: 'font-mono' } } },
  { accessorKey: 'valid_range', header: 'Validation range', meta: { class: { td: 'font-mono' } } },
  { accessorKey: 'baseline', header: 'Current params' },
  { accessorKey: 'candidate', header: 'Candidate' },
  { accessorKey: 'lookahead_ok', header: 'Lookahead' },
  { accessorKey: 'promoted', header: 'Result' },
  { accessorKey: 'note', header: 'Note' },
];

// Newest cycle first
const tableData = computed(() => [...strategyCycles.value].reverse());

function formatTs(ts: string): string {
  const date = new Date(ts);
  return Number.isNaN(date.getTime()) ? ts : timestampms(date);
}

function formatPct(value: number): string {
  return `${value.toFixed(2)}%`;
}

onMounted(load);
</script>

<template>
  <div class="flex flex-col gap-3 px-2 md:px-4">
    <div class="flex flex-row items-center gap-3">
      <h3 class="text-2xl font-bold">Self-Improve</h3>
      <USelect
        v-if="strategies.length > 0"
        v-model="selectedStrategy"
        :items="strategies"
        class="min-w-60"
      />
      <UButton class="ms-auto" icon="i-mdi-refresh" :loading="loading" @click="load">
        Refresh
      </UButton>
    </div>

    <UAlert
      v-if="!loading && cycles.length === 0"
      color="info"
      icon="i-mdi-information"
      title="No self-improvement cycles recorded yet"
      description="The loop appends one line per cycle to self_improve.jsonl. Results show up here once the first cycle has finished."
    />

    <template v-else-if="summary.last">
      <div class="grid grid-cols-2 md:grid-cols-4 gap-3">
        <UCard>
          <div class="text-sm text-neutral-500">Cycles</div>
          <div class="text-2xl font-bold">{{ summary.total }}</div>
        </UCard>
        <UCard>
          <div class="text-sm text-neutral-500">Promoted</div>
          <div class="text-2xl font-bold">{{ summary.promoted }}</div>
        </UCard>
        <UCard>
          <div class="text-sm text-neutral-500">Last cycle</div>
          <div class="text-lg font-bold">{{ formatTs(summary.last.ts) }}</div>
        </UCard>
        <UCard>
          <div class="text-sm text-neutral-500">Last promotion</div>
          <div class="text-lg font-bold">
            {{ summary.lastPromoted ? formatTs(summary.lastPromoted.ts) : '-' }}
          </div>
        </UCard>
      </div>

      <ECharts :option="chartOptions" autoresize :theme="settingsStore.chartTheme" />

      <UTable :data="tableData" :columns="tableColumns">
        <template #ts-cell="{ row }">{{ formatTs(row.original.ts) }}</template>
        <template #kind-cell="{ row }">
          <UBadge :color="row.original.kind === 'code' ? 'primary' : 'neutral'" variant="subtle">
            {{ row.original.kind === 'code' ? 'Code' : 'Params' }}
          </UBadge>
          <div v-if="row.original.kind === 'code'" class="text-sm whitespace-normal max-w-80">
            <span class="font-mono">{{ row.original.variant }}</span>
            {{ row.original.hypothesis }}
          </div>
        </template>
        <template #baseline-cell="{ row }">
          <span
            :class="
              row.original.baseline.profit_pct < 0 ? 'text-(--color-loss)' : 'text-(--color-profit)'
            "
          >
            {{ formatPct(row.original.baseline.profit_pct) }}
          </span>
          <span class="text-neutral-500 text-sm ms-2">
            {{ row.original.baseline.trades }} trades, DD
            {{ formatPct(row.original.baseline.max_drawdown_pct) }}
          </span>
        </template>
        <template #candidate-cell="{ row }">
          <span
            :class="
              row.original.candidate.profit_pct < 0
                ? 'text-(--color-loss)'
                : 'text-(--color-profit)'
            "
          >
            {{ formatPct(row.original.candidate.profit_pct) }}
          </span>
          <span class="text-neutral-500 text-sm ms-2">
            {{ row.original.candidate.trades }} trades, DD
            {{ formatPct(row.original.candidate.max_drawdown_pct) }}
          </span>
        </template>
        <template #lookahead_ok-cell="{ row }">
          <UBadge :color="row.original.lookahead_ok ? 'success' : 'error'" variant="subtle">
            {{ row.original.lookahead_ok ? 'OK' : 'Biased' }}
          </UBadge>
        </template>
        <template #promoted-cell="{ row }">
          <UBadge :color="row.original.promoted ? 'success' : 'neutral'" variant="subtle">
            {{ row.original.promoted ? 'Promoted' : 'Discarded' }}
          </UBadge>
        </template>
        <template #note-cell="{ row }">
          <span class="text-neutral-500">{{ row.original.note || '-' }}</span>
        </template>
      </UTable>
    </template>
  </div>
</template>

<style scoped>
.echarts {
  width: 100%;
  height: 320px;
}
</style>
