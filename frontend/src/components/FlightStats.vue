<script setup lang="ts">
import { useDroneStore } from '../store/drone';

const store = useDroneStore();

function formatTime(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  return `${m}:${s.toString().padStart(2, '0')}`;
}

function batteryColor(pct: number): string {
  if (pct < 50) return '#22c55e';
  if (pct < 80) return '#eab308';
  return '#ef4444';
}

function riskColor(risk: string): string {
  if (risk === 'safe') return '#22c55e';
  if (risk === 'warning') return '#eab308';
  return '#ef4444';
}

function riskLabel(risk: string): string {
  if (risk === 'safe') return '安全';
  if (risk === 'warning') return '警告';
  return '危险';
}

function handleSafeDistanceChange(e: Event) {
  const target = e.target as HTMLInputElement;
  store.droneConfig.safeDistance = Number(target.value);
}

function handleExport() {
  const kml = store.exportPlan();
  if (!kml) return;
  const blob = new Blob([kml], { type: 'application/vnd.google-earth.kml+xml' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'flight-plan.kml';
  a.click();
  URL.revokeObjectURL(url);
}
</script>

<template>
  <div class="bg-slate-800 rounded-lg p-4 space-y-3">
    <h3 class="text-sm font-bold text-slate-200 border-b border-slate-700 pb-2">
      飞行统计
    </h3>

    <div class="grid grid-cols-2 gap-2 text-xs">
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">总距离</div>
        <div class="text-lg font-bold text-sky-400">
          {{ (store.totalDistance / 1000).toFixed(2) }}
          <span class="text-xs text-slate-500">km</span>
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">预计时间</div>
        <div class="text-lg font-bold text-sky-400">
          {{ formatTime(store.estimatedTime) }}
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">电量消耗</div>
        <div class="text-lg font-bold" :style="{ color: batteryColor(store.batteryPercent) }">
          {{ store.batteryPercent.toFixed(1) }}%
        </div>
        <div class="w-full bg-slate-700 rounded-full h-1.5 mt-1">
          <div
            class="h-1.5 rounded-full transition-all"
            :style="{ width: store.batteryPercent + '%', backgroundColor: batteryColor(store.batteryPercent) }"
          />
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">航点数量</div>
        <div class="text-lg font-bold text-sky-400">{{ store.waypoints.length }}</div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">最大高度</div>
        <div class="text-lg font-bold text-sky-400">
          {{ store.waypoints.length > 0 ? Math.max(...store.waypoints.map((w) => w.altitude)) : 0 }}
          <span class="text-xs text-slate-500">m</span>
        </div>
      </div>
      <div class="bg-slate-900 rounded p-2">
        <div class="text-slate-400">算法</div>
        <div class="text-lg font-bold text-purple-400 uppercase">
          {{ store.selectedAlgorithm }}
        </div>
      </div>
    </div>

    <div class="border-t border-slate-700 pt-2">
      <h4 class="text-xs text-slate-400 mb-2">安全设置</h4>
      <div class="space-y-2">
        <div class="flex items-center justify-between text-[11px]">
          <span class="text-slate-400">安全距离</span>
          <span class="font-bold text-sky-400">{{ store.droneConfig.safeDistance }}m</span>
        </div>
        <input
          type="range"
          min="10"
          max="200"
          step="5"
          :value="store.droneConfig.safeDistance"
          @input="handleSafeDistanceChange"
          class="w-full h-1.5 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-sky-500"
        />
        <div class="flex justify-between text-[9px] text-slate-500">
          <span>10m</span>
          <span>200m</span>
        </div>
      </div>
    </div>

    <div class="border-t border-slate-700 pt-2">
      <h4 class="text-xs text-slate-400 mb-2">风险评估</h4>
      <div class="bg-slate-900 rounded p-2 mb-2">
        <div class="flex items-center justify-between">
          <span class="text-[11px] text-slate-400">整体风险</span>
          <span
            class="text-xs font-bold px-2 py-0.5 rounded"
            :style="{ backgroundColor: riskColor(store.routeRisk.overallRisk) + '20', color: riskColor(store.routeRisk.overallRisk) }"
          >
            {{ riskLabel(store.routeRisk.overallRisk) }}
          </span>
        </div>
      </div>
      <div class="grid grid-cols-2 gap-2 text-[10px]">
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500">距禁区最近</div>
          <div class="text-sm font-bold text-slate-300">
            {{ store.routeRisk.minDistanceToZone > 1000
              ? (store.routeRisk.minDistanceToZone / 1000).toFixed(1) + ' km'
              : store.routeRisk.minDistanceToZone.toFixed(0) + ' m' }}
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500">接近禁区点</div>
          <div
            class="text-sm font-bold"
            :style="{ color: store.routeRisk.zoneRiskCount > 0 ? '#eab308' : '#22c55e' }"
          >
            {{ store.routeRisk.zoneRiskCount }}
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2 col-span-2">
          <div class="text-slate-500">地形风险点</div>
          <div
            class="text-sm font-bold"
            :style="{ color: store.routeRisk.terrainRiskCount > 0 ? '#eab308' : '#22c55e' }"
          >
            {{ store.routeRisk.terrainRiskCount }} 个航点低于安全高度
          </div>
        </div>
      </div>
    </div>

    <div class="border-t border-slate-700 pt-2">
      <h4 class="text-xs text-slate-400 mb-1">无人机配置</h4>
      <div class="grid grid-cols-3 gap-1 text-[10px] text-slate-500">
        <div>最大高度: {{ store.droneConfig.maxAltitude }}m</div>
        <div>最大速度: {{ store.droneConfig.maxSpeed }}m/s</div>
        <div>电池: {{ store.droneConfig.batteryCapacity }}mAh</div>
      </div>
    </div>

    <button
      @click="handleExport"
      :disabled="!store.currentPlan"
      class="w-full py-2 rounded text-xs font-medium bg-indigo-700 text-white hover:bg-indigo-600 disabled:opacity-40 disabled:cursor-not-allowed transition"
    >
      导出 KML
    </button>
  </div>
</template>
